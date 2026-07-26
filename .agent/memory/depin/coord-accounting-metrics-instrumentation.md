---
type: decision
tags: [depin, coord, accounting, tally, workertally, monkit, metrics]
created: 2026-07-10
agent: subagent (task-7 accounting instrumentation)
---

Task 7 (commit `6349e9e`, branch `feat/coord-metrics-collection`): monkit
counters/durations added to `coord/accounting/tally` and
`coord/accounting/workertally`, at the real per-cycle completion sites.

**tally** (`coord/accounting/tally/service.go`, `Service.Tally`): added
`mon.DurationVal("tally_duration")`, `mon.IntVal("tally_objects_tallied")`,
`mon.IntVal("tally_bytes_tallied")`, all emitted right after per-cycle totals
are computed (same site as a pre-existing, pre-branch
`monAccounting.IntVal("total_objects"/"total_bytes")` block from the original
Storj-ported accounting system - commit `c1edb12`). That older block was left
untouched; the new metrics deliberately duplicate its semantics under the
plain `mon` scope (vs. the legacy `monAccounting = monkit.ScopeNamed(...)`
scope) for consistency with every other metrics-collection task on this
branch. Flagged as an open judgment call in the task report - a reviewer
could choose to drop the new IntVals and point tests at `monAccounting`
instead.

**workertally** (`coord/accounting/workertally/service.go`, `Service.Run`):
added `mon.Counter("worker_tally_workers_tallied")` (Inc by count of workers
this cycle) and `mon.IntVal("worker_tally_payout_bytes")` (sum of
`TotalBytes` across tallies), emitted right after `SaveWorkerTallies`
succeeds. This package had zero prior monkit instrumentation beyond the
automatic `mon.Task()`.

Both packages already had a package-level `mon = monkit.Package()`; no new
`monkit.go` file was needed anywhere. `coord/accounting/tally/eventkit.go`
was explicitly out of scope and left untouched.

**Tests**: new `monkit_test.go` (per package) with the established
`withTestMon`/`assertCounter`/`durationCount`/`intValLast` test-isolation
helpers (mirrors [[coord-order-metrics-instrumentation]],
[[coord-audit-metrics-instrumentation]],
[[coord-repair-metrics-instrumentation]]), plus `service_metrics_test.go`
exercising the real `Service.Tally`/`Service.Run` against the existing
`fakeStore` fixtures already defined in each package's `service_test.go`
(not mocked monkit internals). `go build`/`go test ./coord/accounting/...`
green, incl. `-race`. Repo-root `go build ./...` has pre-existing unrelated
breakage in `uplink-sdk/cmd/download-demo` and `internal/testplanet`
(verified via `git stash` - not caused by this task).
