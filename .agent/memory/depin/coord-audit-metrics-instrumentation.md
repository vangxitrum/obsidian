---
type: decision
tags: [depin, telemetry, monkit, audit, testing]
created: 2026-07-09
agent: subagent (Task 3 executor)
---

Task 3 of the "coordinator metrics collection" plan (branch
`feat/coord-metrics-collection`, commit `0f04a9f`
"feat(coord/audit): add monkit outcome counters and verify duration"). Purely
domain metrics on `coord/audit` - independent of [[telemetry-remote-write-push]]
(Tasks 1-2); these `mon.Counter`/`mon.DurationVal` calls work with or without a
push client wired up, they just show up on `/metrics` immediately.

## What was added
- `Worker.auditOne` (`coord/audit/worker.go`): after `w.verifier.Verify(ctx, seg)`
  returns (skip-sentinel/hard errors excluded), one `mon.Counter(name).Inc(1)` per
  worker-outcome-instance in the report - `audit_segments_success` (per
  `report.Successes` entry), `audit_segments_failed` (`report.Failures`),
  `audit_segments_offline` (`report.Offlines`), `audit_segments_unknown`
  (`report.Unknown`), `audit_segments_contained` (inside the existing
  `report.PendingAudits` reverify-enqueue loop). Plus `audit_verify_duration`
  (`mon.DurationVal(...).Observe(d)`) measured tightly around the `Verify()` call
  only - still recorded on skip-sentinel errors (Verify() ran), not recorded if
  Verify() is never reached (e.g. segment load failure).
- `ReverifyWorker.processJob` (`coord/audit/reverifyworker.go`):
  `audit_reverify_processed` right before the `switch outcome` (i.e. once a real
  outcome came back from `ReverifyPiece`, whether or not it's binary pass/fail);
  `audit_reverify_pass` in the `AuditSuccess` case; `audit_reverify_fail` inside
  the shared `recordFailure` helper (called from both the `AuditFailure` case and
  the strikes-exhausted `AuditContained` case - one instrumentation site, two call
  sites covered).

## Gotcha: coord/audit already had `var mon = monkit.Package()`
It lives in `coord/audit/observer.go` (in a `var (Error = ...; mon = ...)` block),
not in a dedicated `monkit.go` like `coord/placement`. Adding a new
`coord/audit/monkit.go` with the same var causes `mon redeclared in this block` -
caught by `go build` immediately, deleted the duplicate. **Always grep
`var mon` in the target package before adding a monkit.go file** - not every
package follows the placement.go template of a dedicated file.

## New test-isolation pattern for monkit producer code
No package in this repo previously tested its own `mon.Counter`/`mon.DurationVal`
calls (checked `worker/piecestore`, `coord/rangedloop`, `coord/reputation`,
`coord/relay` - none assert on package-level `mon` output). The existing
`monkit.NewRegistry()` usages (`pkg/telemetry/client_test.go`,
`pkg/debug/server_test.go`) test *consumers* of monkit stats (the push client,
the debug server) via a fake `monkit.StatSourceFunc` - a different problem.

Established pattern for testing a *producer* package var: since the test file is
in the same package, reassign the package-level `mon` var directly - no
interface/DI needed in production code.

```go
func withTestMon(t *testing.T) *monkit.Scope {
    t.Helper()
    prev := mon
    scope := monkit.NewRegistry().ScopeNamed("audit_test")
    mon = scope
    t.Cleanup(func() { mon = prev })
    return scope
}
```
Then assert with `scope.Counter(name).Current()` (exact int64) or
`scope.DurationVal(name).Stats(cb)` reading the `"count"` field back out (no
public `.Count()` accessor on `DurationVal`). Caveat: tests using this must not
`t.Parallel()` against each other since `mon` is one shared package var. See
`coord/audit/monkit_test.go`. Reusable for any future monkit-producer
instrumentation task in this repo (e.g. if Tasks 4+ add counters elsewhere).

## Judgment calls flagged in the task report (not blocking, but worth knowing)
1. `audit_reverify_processed` fires on all 6 real `processJob` outcomes
   (success/failure/client-fault/contained-below-limit/contained-exhausted/
   offline-or-unknown-retry), not just the two that resolve pass/fail - "processed"
   means "a real outcome came back," not "was resolved."
2. The `audit_segments_*` counters increment once per worker-outcome-instance in a
   report (a single segment audit can report multiple workers, one per erasure
   share), not once per `auditOne` call - matches the existing
   `AuditStats.Update` precedent in `coord/placement/audit.go` which sums
   `len(report.Successes)` etc., despite the metric names reading "segments."
