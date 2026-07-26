---
type: fact
tags: [depin, coord, file, monkit, metrics, testing]
created: 2026-07-09
agent: task-6-subagent
---

Task 6 (commit `3670b02`, branch `feat/coord-metrics-collection`, same branch
as [[coord-order-metrics-instrumentation]], [[coord-audit-metrics-instrumentation]],
[[coord-repair-metrics-instrumentation]]): added monkit counters for
create/commit/download outcomes in `coord/file/endpoint.go`.

**Metrics added:**
- `file_created` / `file_created_bytes` (IntVal, `Observe(req.GetSize_())`) -
  in `CreateFile`, right after `e.service.Create(...)` succeeds. No
  inline/remote tag: at file-creation time no segment exists yet, so the kind
  isn't known (decided later, per upload).
- `segment_committed` / `segment_committed_bytes` - tagged
  `kind=remote` (in `CommitSegment`, after `e.service.CommitSegment` succeeds,
  bytes = `req.SizeEncryptedData`) or `kind=inline` (in `MakeInlineSegment`,
  after `e.service.MakeInlineSegment` succeeds, bytes =
  `len(req.EncryptedInlineData)`).
- `segment_downloaded` / `segment_downloaded_bytes` - tagged
  `kind=remote`/`kind=inline` in `buildDownloadSegment` (the one shared helper
  both `GetDownloadInfo` and `GetDownloadInfoByTicket` call through
  `buildManifest` - single instrumentation site for both RPCs, same "shared
  site" precedent as Task 5's `Signer.Sign`), at each branch's success point
  (inline: right before the early return; remote: right after
  `CreateGetOrderLimits` succeeds).

**Key mapping decision:** the brief said "create/commit/download" without
line numbers. `coord/file/endpoint.go` has 9 RPCs, not 3. Resolved by: (1)
`BeginSegment` cannot be "create" - its own doc comment says nothing is
written to the DB there (CommitSegment/MakeInlineSegment do the actual
persist). (2) A segment's only DB-write event is `CommitSegment` (remote) or
`MakeInlineSegment` (inline) - no separate create-then-commit two-phase exists
for segments in this codebase - so "commit" = that pair, broken down by kind.
(3) That leaves `CreateFile` (which does write a DB row via
`Service.Create` -> `s.store.Create`) as "create" by elimination and by
matching RPC name; it just doesn't get a kind tag since none exists yet.

**Tag reuse (per task instruction, not re-derived):**
- Download: reused `buildDownloadSegment`'s existing
  `len(seg.InlineData) > 0` branch (already sets `ds.Kind` to
  `*pb.DownloadSegment_Inline`/`_Remote` - the pb oneof field is itself named
  "Kind", hence the monkit tag key `"kind"`).
- Commit: this codebase already splits segment persistence into two separate
  RPCs by kind (`CommitSegment` for remote/CLONE, `MakeInlineSegment` for
  inline) - that structural split IS the existing distinction, reused
  directly rather than re-derived from a field check (none exists at either
  site).

**Gotcha found but left as a disclosed limitation:** `Service.CommitSegment`/
`MakeInlineSegment` are idempotent (retried commit for an already-persisted
segment short-circuits without a new DB write), but the endpoint has no way
to observe "real write vs. idempotent no-op" (`Service.*` returns only
`error`). My counters fire on every successful `Endpoint.CommitSegment`/
`MakeInlineSegment` call, so a client retry of an already-committed segment
double-counts by one. Fixing needs a `service.go` signature change, out of
the brief's `endpoint.go`-only file scope - flagged in the task report rather
than fixed.

**Test pattern:** reused [[coord-audit-metrics-instrumentation]]'s
`withTestMon`/`assertCounter`/`intValSum` helpers (ported verbatim from
`coord/order/monkit_test.go`) in a new `coord/file/monkit_test.go`. Tests in
`coord/file/endpoint_metrics_test.go` drive the *real* `Endpoint.CreateFile` /
`CommitSegment` / `MakeInlineSegment` / `buildDownloadSegment` - not mocks -
against a real `overlay.Service` (in-memory placements map is enough,
`Placement()` is just a map lookup), a real `overlay.IdentityCache` /
`overlay.DownloadSelectionCache` (both wrap a narrow, easily-fakeable source
interface: `workerIdentitySource.GetWorkersByIDs`,
`downloadReader.SelectDownloadableWorkers` - NOT the cache types themselves),
and a real `order.Service` (its `Store` interface is literally `interface{}`,
so `nil` satisfies it - `CreateGetOrderLimits`/`CreatePutOrderLimits` never
touch it, only `service.coord` the signer). Peer identity injected via
`grpcpeer.NewContext(ctx, &grpcpeer.Peer{State: tls.ConnectionState{
PeerCertificates: ident.Chain()}})`, same pattern as
[[coord-order-metrics-instrumentation]]'s `settlement_metrics_test.go`. Using
CLONE algorithm (`RedundancyScheme.PieceSize` returns `encryptedSize` as-is
for CLONE) sidesteps Reed-Solomon stripe-size arithmetic entirely for these
tests - worth remembering for any future `coord/file` test needing a
minimal-but-real redundancy scheme.

All `coord/file` tests green (build/vet/gofmt/goimports/`test -race -v` all
clean, including all pre-existing tests). `go build ./...` still shows only
the 2 pre-existing unrelated failures (`uplink-sdk/cmd/download-demo`,
`internal/testplanet`) already known to predate this branch.

## Fix follow-up (commit `2f375c7`, same branch)

The disclosed idempotent-retry double-counting gotcha above got a reviewer
finding, and the human explicitly approved crossing into `service.go` to fix
it (the original brief's file-scope restriction was for the initial task,
not the fix). Resolution:

- `Service.CommitSegment`/`Service.MakeInlineSegment` signatures changed to
  `(wasNew bool, err error)`. `wasNew` is `true` only when a fresh segment row
  was actually persisted via `store.WithTx`/`CreateSegment`; `false, nil` on
  the pre-existing idempotent no-op path (segment already existed).
- `endpoint.go`'s two call sites now gate the `segment_committed`/
  `segment_committed_bytes` increments on `if wasNew`. RPC success/response to
  the client is unchanged either way - purely a metrics-accuracy fix.
- Grepped whole repo for `.CommitSegment(`/`.MakeInlineSegment(` first - only
  Go callers of the *service* methods were the two endpoint handlers (other
  matches: generated gRPC dispatch, uplink-sdk's client-side batcher calling
  the *RPC* not the service, and this package's tests). No surprise call
  sites.
- `coord/file/service_test.go`'s `fakeStore` made properly stateful:
  `CreateSegment` now appends to `f.segments`, `GetSegmentByFileIDAndNumber`
  now searches it by `FileId`+`SegmentNumber` instead of hardcoding
  `vo.ErrRecordNotFound`. This is what let the new retry test actually
  exercise the idempotent path (previously always-not-found made that
  impossible to hit). Backward compatible - no existing test relied on the
  hardcoded not-found behavior.
- New test `TestSegmentCommitted_RetryDoesNotDoubleCount`
  (`coord/file/endpoint_metrics_test.go`): calls `Endpoint.CommitSegment` and
  `Endpoint.MakeInlineSegment` each twice with an identical signed
  `SegmentId`/request, asserts the counter lands at 1 (not 2) after the
  retry, and that both calls still return success to the client.
- Full verification green: `go build ./coord/file/...`,
  `go test ./coord/file/... -race -count=1 -v` (all pass, no failures),
  `go build ./coord/...` (confirms no other caller broke).
- The "missing segment_created counter" reviewer finding from the same review
  round was explicitly left alone per the human's decision (`file_created` as-is,
  no code change) - not touched by this fix.
