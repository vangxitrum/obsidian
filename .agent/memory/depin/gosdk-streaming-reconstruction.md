---
type: decision
tags: [depin, go-sdk, download, streaming, reed-solomon, edge, throughput]
created: 2026-08-03
agent: main
---

# Go SDK stripe-level streaming reconstruction

The Go SDK now feeds live worker piece readers directly into the existing
`eestream.DecodeReaders2` pipeline for Reed-Solomon segments. It no longer waits
for `piecedownload.Manager.Fetch` to materialize the required pieces as full
`[]byte` buffers before decoding. Serial processing between segments remains.

**Why:** The old eager piece buffering forced edge downloads into two serial
network legs: worker-to-edge followed by edge-to-HTTP-client. Streaming lets the
decoder emit plaintext once K shares of the first stripe arrive, overlapping
worker ingress, RS reconstruction, decryption, and HTTP egress.

## Design

- `Manager.OpenReaders` creates a lazy reader for every candidate but admits at
  most `required + DefaultLongTail` active streams (K+6 by default).
- The client-global limiter atomically reserves the whole active window for a
  segment. This prevents concurrent segments from each holding fewer than K
  permits and deadlocking reconstruction.
- The global reservation remains owned by the segment until all candidate
  readers close. Failed active readers release only their local window position,
  so reserve candidates reuse the segment's existing global permits and cannot
  deadlock behind a queued full-window reservation.
- Successful EOF retains its local position until decoder close, preventing
  completed short ranges from cascading through every reserve holder.
- `WithDownloadConcurrency` is effectively at least K. A configured global
  capacity below K returns an explicit error because streaming reconstruction
  cannot operate below the required share count.
- Parallel piece-read timing uses union wall time through
  `metrics.Collector.ConcurrentReader`, not the sum of overlapping worker waits.
- The Storj pending-share cleanup fix was mirrored into both eestream copies.
- CLONE still uses the eager `Fetch` path to preserve its existing fastest-copy
  failover behavior; this change targets stripe-based Reed-Solomon reconstruction.

## Verification

- A race-tested segment regression deliberately blocks every contributing piece
  halfway and proves plaintext arrives before any piece completes.
- Tests cover K+6 active-window bounds, reserve replacement, successful EOF not
  activating reserves, atomic cross-segment admission, and the queued-reservation
  replacement deadlock.
- Focused SDK race tests pass for root, metrics, piecedownload, segmentdownload,
  and eestream.
- DePIN testplanet, edgeserver, and eestream tests pass; edgeserver builds.
- Full SDK `./...` still has only the unrelated known
  `internal/vo.TestPieceIDScanNullAndEmpty` failure.

## Live result

The deployed edge was confirmed to contain this implementation by its new
`metrics.go:291` transfer log format and overlapping-download metric note.

- Local new-SDK 8x80MiB: 8/8, 79s, 8.10MiB/s aggregate.
- First deployed-edge 8x80MiB: 8/8 exact bodies, 121s, 5.29MiB/s aggregate;
  requests completed in 111.39-120.63s.
- Immediate edge repeat: 7/8, 111s, 5.05MiB/s successful-byte throughput;
  successful requests completed in 99.08-111.74s.
- Previous eager edge baseline: 8/8, 142s, 4.51MiB/s, 137.15-141.41s each.

Streaming therefore improved observed aggregate edge throughput by roughly
12-17% and materially reduced request times, but did not close the gap to direct
SDK throughput.

The repeat's one failure was an admission timeout, not a piece/decode failure:
HTTP 504 at 50.20s and edge log `reserve stream window: context canceled`. The
default global cap 280 holds exactly eight K+6 windows (`8 * 35`), so any normal
concurrent edge request makes one benchmark request wait for a full window until
the upstream HTTP context cancels. Edge needs configurable admission headroom
(for example one or two additional 35-stream windows) or an HTTP queue timeout
longer than the proxy timeout before this scheduler is production-safe at eight
benchmark requests plus background traffic.

Follow-up: edge now exposes `server.global-download-concurrency` and defaults it
to 512, explicitly passing `WithGlobalDownloadConcurrency(512)` to its SDK
client. This admits fourteen complete 35-stream windows (490 streams, 22 spare)
and matches the edge's default libp2p outbound stream ceilings. The SDK-wide
default remains 280 for other consumers. Edge race tests, binary build, config
flag/default inspection, and diff checks passed; live validation requires
redeploying the edge and repeating the benchmark.

Post-redeploy 512-cap validation (`dl-streaming-cap512-20260803`): the edge
container changed and relay circuits reset before the run; 8x80MiB then completed
8/8 with exact 83,886,080-byte HTTP 200 bodies, 126.21-135.23s per request,
135s wall, and 4.74MiB/s aggregate. Loki had zero `reserve stream window` and
zero `download failed` entries during the run. The cap increase therefore fixed
the admission failure. It did not improve the network throughput ceiling: this
sample was below the prior streaming run's 5.05-5.29MiB/s, though still above
the eager 4.51MiB/s baseline. Relay reported no additional circuit rejections.

Related: [[download-network-cap-root-cause]], [[gosdk-segment-prefetch]],
[[segment-prefetch-quiescence-bug]].
