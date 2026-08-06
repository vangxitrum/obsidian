---
type: fact
tags: [depin, go-sdk, edge, download, eestream, prefetch, bug]
created: 2026-08-02
agent: main
---

**Root cause of "every 256 MiB edge download fails": the eestream inactivity
watchdog counts segment-prefetch backpressure as a stall.**

Found 2026-08-02, immediately after edgeserver log shipping went live (see
[[download-failure-fixes-applied]]). The deployed edge logged:

```
msg="download failed"  error="sdk: stream segment 2: quiescence"
```

`quiescence` is `ErrInactive` from `go-sdk/pkg/eestream/common.go:17`. Its
watchdog lives in `pkg/eestream/stripe.go:140-172`: every
`inactiveCheckInterval` (1s) it snapshots per-piece progress, and after
`inactiveCheckMaxCount` (5) *identical* snapshots - i.e. **5 seconds with no
per-piece progress** - it sets `s.inactive`, which makes the next stripe read
return `ErrInactive` (stripe.go:357).

The watchdog goroutine is started as soon as the share readers are launched,
i.e. **at segment open, not at first read**. `DefaultSegmentPrefetch = 2` means
segment 2 is opened while segment 1 is still being copied to the client. Segment
2 fills its buffers, then legitimately stops making progress - nobody is reading
it yet - and after 5 s the watchdog declares it dead. So the download always
delivers exactly segment 1 and then fails on segment 2.

**Why only the edge, and only multi-segment files.** It is a race:
quiescence fires when (time left to finish segment 1) - (time to fill segment
2's buffers) > 5 s. From this workstation the whole 4-segment 256 MiB download
takes ~41 s and segment 1 ~10 s, so the gap rarely reaches 5 s and
`DownloadByTicket` completes all 268435456 bytes. On the edge (2.2 MB/s,
segment 1 alone ~20-29 s) the gap is always well past 5 s, so it fails every
time. Single-segment files (30 MiB) never prefetch a second segment and always
work.

Every earlier observation fits: always exactly 67106816 bytes delivered
(one segment; the 2048 shortfall is just in-flight data lost when the abort
drops the connection), wall time varying 17-29 s with an identical byte count
(so NOT a timeout), and small Range requests into segments 2 and 4 succeeding
(a range-scoped manifest covers one segment, so nothing is prefetched).

**FIXED 2026-08-02** (option 1, the real fix). Added
`StripeReader.awaitingStripe atomic.Bool`, set around the consumer's
`stripeReady.Wait(ctx)` in `ReadStripes`. The watchdog now resets `match` when
`!awaitingStripe`, so quiescence means "no progress **and** someone is waiting
for bytes" rather than "buffers are full". Healthy backpressure from an
unread prefetched segment no longer counts.

Two regression tests, RED first, in both copies:
- `TestDecodeReaders2_IdleConsumerIsNotMistakenForAStall` - >maxStripesAhead
  (256) stripes so the readers park on backpressure, consumer sleeps past the
  whole inactivity window, then reads. Before the fix it failed with exactly the
  production error (`quiescence` at `ReadStripes:358`).
- `TestDecodeReaders2_StalledPiecesAreStillDetected` - readers that deliver
  nothing while the consumer waits must STILL produce quiescence, proving the
  watchdog's real purpose survives. Without this second test the "fix" could
  silently disable stall detection.

Both pass under `-race` in `go-sdk/pkg/eestream` and `depin/pkg/eestream`.
Rejected alternatives: `WithSegmentPrefetch(1)` only hides it and loses the
overlap; raising `inactiveCheckMaxCount` just widens the race.

NOTE the watchdog is duplicated: `go-sdk/pkg/eestream/` and
`depin/pkg/eestream/` both define `ErrInactive` and the same logic - fix both.
