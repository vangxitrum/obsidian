---
type: decision
tags: [go-sdk, download, retry, resume, eestream, edgeserver, coverage]
created: 2026-08-14
agent: main
---

# Transparent mid-stream download retry (go-sdk + edge)

Ported Storj uplink's Layer-3 retry (`uplink/private/stream/download.go`) into
go-sdk, plus edge-side Content-Length and resume. Uncommitted on go-sdk `main`
and depin `develop`.

## The blocker nobody would have found without looking

`go-sdk/pkg/eestream/stripe.go` `combineErrs` formatted each piece's error into
a **string with `%v`** instead of upstream's `errs.Group` + `%w`. That is the
error returned on the single most common mid-stream failure ("ran out of live
pieces"), so **every leaf error was invisible to `errors.Is`/`errors.As`** and
no retry policy above eestream could ever have fired. Fixed to match upstream,
plus a new `ErrPiecesUnavailable` class (upstream uses the generic `Error`, but
that class is also used for programmer errors like "negative expected size",
which must not consume a retry budget). Guarded by
`pkg/eestream/combine_errs_test.go`.

## Design

- Retry loop lives in `download_retry.go`, wrapping the segment loop rather
  than porting uplink's pull-based `Download` io.ReadCloser: `io.Copy` already
  returns bytes-written-to-`w` even on error, so the resume offset is handed to
  us instead of reconstructed.
- `ResolveTicketRange` + `DownloadManifestRange` already existed and ARE the
  `resetReader` primitive. First attempt is whole-file; **every retry is ranged**
  (one-way switch) because an unranged manifest leaves `PlainOffset == 0` on
  every segment.
- `maxDownloadRetries = 6` shared across decryption/quiescence/network, reset on
  progress, plus `maxTotalDownloadRetries = 64` that never resets (upstream has
  the unbounded-loop hole; this closes it).
- `runDownloadAttempts` takes an injectable attempt func so the policy is unit
  testable without a coordinator.
- `segmentdownload.ErrorDetection{Enabled,Offset}` threaded through
  `Open`/`OpenRange`. **`pkg/eestream` needed no change** - `DecodeReaders2`
  already took `forceErrorDetection`, it was just hardcoded false.

## Traps that bit

- `status.Code(err)` returns `codes.Unknown` for ANY non-gRPC error, so matching
  on it classified every ordinary error as retryable. Use `status.FromError` and
  require ok; `Unknown` is deliberately NOT retryable.
- The SDK stall timer (`DefaultDownloadTimeout`, 2m inactivity) cancels a ctx
  **shared by every attempt**. Guard on `ctx.Err()` first or one stall burns the
  whole budget instantly and fires 6 useless coordinator resolves.
- `io.Copy` cannot distinguish source from sink failure. `latchingWriter` records
  the writer's last error so a hung-up HTTP client is never retried.
- Edge: `cw.written` is post-gzip WIRE bytes and `cz.inBytes` misses sub-MinSize
  buffered bytes - **neither is a valid resume offset**. Added
  `plainCountingWriter` above compression. It must embed `http.ResponseWriter`
  or `http.ResponseController`'s Unwrap walk stops there and Flush silently
  no-ops (this deadlocked the truncation test).
- Coord rejects any range whose offset is not below file size, so a 0-byte file
  416s on `{0,1}` AND `{0,0}`. Added `Client.ResolveTicket` (Range: nil) - the
  only way to resolve an empty file. `serveFull` uses it, giving Content-Length
  for zero extra RPCs.

## e2e: making it genuinely RED

`internal/testplanet/download_retry_test.go`. Killing holders down to
`RequiredShares` mid-stream is NOT red - erasure coding alone absorbs it. To
reproduce the real failure you need all three:
1. `WorkerCount == TotalShares`, or each segment picks its own subset and killing
   one segment's holders says nothing about the rest.
2. `cfg.Order.DownloadTailToleranceOverrides = "2-2"` so coord signs exactly
   RequiredShares limits and the in-flight attempt has no spare to fail over to.
3. Mark killed workers offline **via `last_seen`, not `is_online`** (download
   eligibility is window-based and never reads `is_online`) + call
   `coord.Overlay.Download.Refresh(ctx)`. Otherwise coord re-signs the same dead
   workers on all 6 retries. **This staleness window is real production
   behaviour: a client cannot retry its way out of a failure faster than the
   coordinator notices it.**

Verified RED 3/3 with retries disabled, GREEN 4/4 enabled.

Related: [[gosdk-streaming-reconstruction]], [[segment-prefetch-quiescence-bug]],
[[stripe-streaming-review]], [[gosdk-coverage-70]].
