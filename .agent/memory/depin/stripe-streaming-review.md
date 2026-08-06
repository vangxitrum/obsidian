---
type: fact
tags: [go-sdk, download, streaming, eestream, limiter, accounting, review]
created: 2026-08-03
agent: main
---

Review of the uncommitted go-sdk stripe-level streaming reconstruction found two scheduler defects: `OpenReaders` starts a goroutine for every holder, so EOF releases from the reserved K+6 readers let all queued candidates fetch (especially for short range windows), and silent reserved readers are never evicted to admit healthy reserve holders. Quiescence therefore fails the segment rather than retrying as Storj's `private/stream/download.go` does.

Re-review after adding `pieceWindow` found that the hard active-call cap is now enforced, but both semantic defects remain: successful EOF releases a window slot and starts an unnecessary reserve candidate, while a silent stream never releases its slot and quiescence still fails the segment. The active-window test covers explicit `Close`, not EOF or silent-stream behavior. The new concurrent wall timer also commits its current union interval only when the final active `Read` returns; `decodedReader.Close` does not wait for stripe-reader goroutines, so transfer metrics can snapshot before cancellation-driven reads finish and omit the final interval.

Final re-review: successful EOF now correctly retains its per-segment slot, and concurrent metric snapshots include active intervals. One new deadlock remains between atomic global reservations and failed-stream replacements: `semaphore.Weighted` is FIFO, so a later segment's full-window `Acquire(N)` can sit ahead of the current segment's replacement `Acquire(1)`. The current segment's healthy readers then park on eestream backpressure while retaining the N-1 global slots needed by the full-window waiter, leaving neither request able to proceed until timeout. No test covers a queued full-window reservation racing a replacement.

Final segment-lifetime design resolves that deadlock: one full-window global reservation is held until every candidate reader for the segment closes, and both initial readers and replacements bypass further global acquisition while remaining bounded by the per-segment window. `TestOpenReaders_ReplacementReusesSegmentReservation` covers a failed stream replacement while a second segment's full reservation is queued. Final focused race tests and mirrored eestream tests passed with no new findings; silent-stream replacement remains an acknowledged limitation.

The accompanying progressive download-order implementation is incompatible with the current worker endpoint: the SDK sends later order frames, but `worker/piecestore/endpoint.go` receives only the opening frame and settles that opening order. Large piece downloads therefore settle at the initial 1 MiB authorization even when the worker streams the full requested length.

The mirrored uncommitted `depin/pkg/eestream/stripe.go` release-drain patch matches Storj commit `607525ba` and had no review finding. Focused race tests passed; the full go-sdk suite only failed the known unrelated `internal/vo.TestPieceIDScanNullAndEmpty` test.
