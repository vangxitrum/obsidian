---
type: fact
tags: [depin, coord, gc, monkit, metrics, testing]
created: 2026-07-10
agent: main
---

Task 10 (commit fa93b32, branch feat/coord-metrics-collection): monkit
instrumentation for coord/gc/bloomfilter and coord/gc/sender.

- Both sub-packages already had `var mon = monkit.Package()` (in observer.go /
  service.go respectively) - no new monkit.go needed.
- `gc_bloomfilter_bundles_built` (Counter), `gc_bloomfilter_pieces_per_filter`
  (IntVal), `gc_bloomfilter_filter_size_bytes` (IntVal) recorded in
  `Observer.Finish` (coord/gc/bloomfilter/observer.go) right after the
  `filters` map is built from `o.master` - every fork already `Join`'d, no
  filter's contents change again after this point - and deliberately *before*
  the `storage.Upload` call, i.e. not gated on upload success (these measure
  build completion, a separate concern from the storage hand-off).
  `Filter.Size()` (pkg/bloomfilter) returns the exact byte length of
  `Filter.Bytes()`, used directly for the size metric.
- `gc_sender_retain_success` / `gc_sender_retain_fail` (Counters) recorded at
  the Retain RPC's actual outcome in coord/gc/sender/service.go. `Service.send`
  previously inlined `piecepb.NewPieceStoreServiceClient(conn).Retain(...)`;
  split the call + outcome recording into a new unexported
  `retain(ctx, client piecepb.PieceStoreServiceClient, info) error` helper so
  tests can substitute a fake client at the pre-existing generated
  `piecepb.PieceStoreServiceClient` interface boundary, without any real dial.
  No new interface type introduced, no change to `dial.Dialer` or
  `NewService`'s signature - `send`'s dial/peer-URL-construction failures are
  deliberately excluded from the counters since they never reach the RPC.
  Rationale mirrors coord/repair/repairer's `Uploader`/`fakeUploader` pattern:
  `dial.Dialer` is a concrete struct requiring real mTLS, so every RPC-calling
  service in this repo that has tests mocks at an interface boundary, never a
  real dial.
- Confirms the repo-wide monkit test-isolation pattern is now used
  consistently in 7+ subsystems: per-package `monkit_test.go` with
  `withTestMon(t) *monkit.Scope` (swap package `mon` for a fresh
  `monkit.NewRegistry().ScopeNamed(...)`, restore via `t.Cleanup`; not
  parallel-safe since `mon` is one shared package var) plus
  `assertCounter`/`intValSum`/`intValCount` helpers reading monkit's
  `Stats(func(key, field, val))` "sum"/"count" aggregation fields.
- Metric naming convention reconfirmed: `<subsystem>_<noun>_<outcome>`,
  separate named counters per outcome (mirrors `repair_success`/
  `repair_failed`, `audit_segments_success`/`_failed`) rather than one
  tagged counter, since there's no natural cross-cutting tag dimension here
  (unlike order's `action` tag).
- Full task report: `.superpowers/sdd/task-10-report.md` in the depin repo
  (this was worked from a treehouse worktree at
  `/home/tuan/.treehouse/depin-b971d9/1/depin`, not the primary
  `/home/tuan/work/depin-workspace/depin` checkout - same branch
  feat/coord-metrics-collection).

See also [[coord-repair-metrics-instrumentation]], [[coord-audit-metrics-instrumentation]], [[coord-order-metrics-instrumentation]], [[coord-overlay-metrics-instrumentation]].
