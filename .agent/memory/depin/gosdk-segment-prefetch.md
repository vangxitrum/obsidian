---
type: decision
tags: [go-sdk, download, prefetch, throughput, edge, segments, ws3]
created: 2026-07-29
agent: main
---

**SUPERSEDED 2026-08-03: go-sdk segment read-ahead / prefetch was removed.**

`WithSegmentPrefetch`, `DefaultSegmentPrefetch`, option/client state, and the
concurrent look-ahead pipeline were removed. `streamSegments` now strictly opens,
copies, and closes one segment before opening the next. Prefetching full pieces
increased memory and piece fan-out without fixing the edge's two-leg
store-and-forward throughput limit. The intended fix is stripe-level streaming
reconstruction, which overlaps worker ingress with HTTP egress without opening
future segments.

Serial behavior is guarded by `download_segments_test.go`; the old
`download_prefetch_test.go` was deleted. Focused SDK race tests, edge/testplanet
tests, and the edge build passed. Full SDK tests still have the unrelated known
`internal/vo.TestPieceIDScanNullAndEmpty` failure.

The text below is retained as historical context only.

**Original WS3 decision: go-sdk segment read-ahead / prefetch.**
Implements `2026-07-29-edge-download-speed.md` workstream 3 (the isolated,
WS4-independent one). go-sdk changes UNCOMMITTED at
`/home/tuan/work/depin-workspace/go-sdk`; depin e2e test uncommitted too.

## Problem
`download.go` looped segments strictly serially: `Open(seg)` -> `io.Copy` ->
`Close` -> next. Segment N+1's piece fetch (network/latency-bound over the
relay) didn't start until N fully streamed, so multi-segment objects paid each
segment's fetch latency back to back. Only helps multi-segment (VOD/large)
objects; single-segment small HLS chunks (~480 KB) unaffected.

## What shipped (go-sdk)
- `options.go`: `WithSegmentPrefetch(depth int) ClientOption` + `segmentPrefetch`
  field on `options`.
- `client.go`: `Client.segmentPrefetch` field, wired in `newClient` (pure copy,
  matching the downloadConcurrency idiom); `TestNewClient_CopiesAllOptionsFields`
  updated (this test is the guard against forgetting to copy an options field -
  every new option MUST be added there).
- `download.go`:
  - `const DefaultSegmentPrefetch = 2`; `(c *Client) prefetchDepth()` applies the
    default when unset/<=0 (depth 1 = old serial behavior).
  - `segmentJob{number, open func(ctx)(io.ReadCloser,error)}` + `streamSegments(...)`
    helper: bounded read-ahead. Opens up to `prefetch` segments ahead
    concurrently, holds their reader handles, `io.Copy`s to `w` in ascending
    segment order (ordering preserved regardless of open-completion order). One
    buffered result channel per job; window slides (launch next after each
    segment finishes copying). On first error (open/stream/close) it cancels a
    child ctx to abort in-flight look-ahead opens, drains + closes already-opened
    readers (no leak), returns wrapped err. Memory bound ~= depth x segment size.
  - Both `DownloadManifest` (full) and `DownloadManifestRange` (range) rewritten
    to build `[]segmentJob` then call `streamSegments`. Range loop keeps the
    per-segment intersection math (skip disjoint, compute segWantStart/End +
    pieceOffset) up front, captured in the closure. Error messages preserved
    exactly via an `errWrap(number, stage, err)` param (" range" suffix for the
    range loop).

## Key facts / decisions
- Concurrency-safe by construction: `metrics.Collector` is mutex-guarded (its
  `Add`/`Timer`/`Reader` all lock), and `wc := workerclient.New(dialer,...)` +
  the dial pool are already built for concurrent piece fetches, so running up to
  `depth` `segmentdownload.Open` calls concurrently against a shared `coll`/`wc`
  is safe. Verified with `go test -race`.
- Opening a segment ahead only kicks off its PIECE FETCHES (buffers fetched
  pieces); RS-decode+decrypt stay pull-driven by reads on the returned reader, so
  look-ahead overlaps fetch with serve without eagerly decoding - the exact
  overlap the plan wanted ("io.ReadCloser handles in an ordered ring").
- Did NOT reuse the existing `collectOrdered` (download_concurrency.go): it's
  currently DEAD non-test code that materializes each item fully into a `[]byte`
  (would buffer a whole 64 MiB segment per index). The reader-streaming
  `streamSegments` is the right tool; left `collectOrdered` untouched (it has its
  own tests).
- Default depth 2 is conservative: with unlimited download concurrency it means
  ~depth x k concurrent piece streams over the relay. Interacts with WS2 (per-conn
  cap) but opt1 single-use relay circuits (commit 297433a) mean more concurrent
  circuits = more relay load, not resets. Fine as default.

## Tests (all green)
- go-sdk unit `download_prefetch_test.go` (package uplinksdk, `-race` clean):
  ordering-out-of-order-completion, prefetch-window bound (max concurrent opens
  <= depth), open-error stops+closes-all (leak check), copy-error->"stream" wrap,
  cancel-look-ahead-on-error (promptness), depth-1-serial, empty-jobs,
  prefetchDepth default/override/negative.
- depin e2e `internal/testplanet/segment_prefetch_test.go`
  `TestSegmentPrefetchDownload`: `Reconfigure.Coord` sets
  `cfg.File.DefaultSegmentSize = 16*1024` (above the 4 KiB inline threshold ->
  all remote), uploads 7*segSize+1234 -> 7+ remote segments, asserts whole-file
  + 7 ranges (incl. boundary-crossing, spanning-3-segments, to-EOF) reconstruct
  exact bytes. PASSED (2.5s). Existing `TestRangedDownload`/`TestEdgeRangedDownload`
  regression-clean against the rewrite (24.7s).

## e2e wiring gotcha (durable)
depin go.mod pins go-sdk to a pinned pseudo-version
(`replace aioz-depin/go-sdk => gitlab.internal/aioz-depin/go-sdk v0.0.0-...f9cbaeeb4a4e`),
NOT the local sibling. To run the e2e against uncommitted go-sdk changes: temporarily
`replace aioz-depin/go-sdk => ../go-sdk` (sibling dir resolves directly, no symlink
needed here), run with `DEPIN_TEST_POSTGRES="postgresql://admin:admin123@localhost:5445/hub?sslmode=disable"`,
then restore go.mod/go.sum. Postgres was up on :5445. LSP shows go-sdk symbols as
"undefined" (separate module not in gopls workspace) - trust `go build`/`go test`,
not the LSP (same as [[gosdk-ranged-download]]).

## Follow-ups
- FINALIZE: user commits+pushes go-sdk (per workflow, user does go-sdk commits),
  then re-pin depin go.mod to the new pseudo-version. Until then WS3 is NOT active
  in depin builds.
- WS2 (bounded relay conn reuse) still gated on WS4 (live rcmgr reset-threshold
  measurement) - both deferred, need user's live relayed stack. WS2 re-touches the
  just-fixed single-use-relay regression, unsafe to ship without WS4's M.

Related: [[grpc-pool-reuse-relay-regression]], [[edge-download-timeout-fix]],
[[gosdk-ranged-download]], [[edge-http-range]].
