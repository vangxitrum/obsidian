---
type: decision
tags: [depin, coord, metrics, monkit, prometheus, grafana]
created: 2026-07-10
agent: claude (task-11 subagent)
---

Task 11 (final task, branch `feat/coord-metrics-collection`): wrote
`monitoring/README.md` + 3 Grafana dashboards (`coord-overview.json`,
`coord-storage-lifecycle.json`, `coord-integrity.json`) under
`monitoring/grafana/dashboards/`. Pure static artifacts for a separate
monitoring VPS - nothing in the depin repo runs them, no compose files
touched.

**Two field-name corrections found by reading the pinned monkit source**
(`github.com/spacemonkeygo/monkit/v3@v3.0.25-0.20260113195619-706ad4b46206`),
important for anyone writing more PromQL against this pipeline:

1. `monkit.Counter` (`mon.Counter(name).Inc(n)`) emits Prometheus label
   `field="value"` (the running total), **not** `field="count"`. Also emits
   `field="high"`/`field="low"` watermarks. Confirmed against
   `coord/audit/monkit_test.go`'s `assertCounter` (reads
   `scope.Counter(name).Current()`, the same value as the "value" field).
2. `IntVal`/`DurationVal`/`FloatVal` (backed by `IntDist`/`DurationDist`/
   `FloatDist`) emit fields `count, sum, min, max, ravg, recent` (plus
   reservoir quantiles `r10/r50/r90/r99/rmin/rmax`, dropped by default via
   `AIOZ_METRICS_DROP_QUANTILES=true`). The "average" field is **`ravg`**,
   not `avg` - there is no `avg` field. `pkg/telemetry/config.go`'s own
   comment says this explicitly. `count`/`sum`/`min`/`max` are all-time
   since construction/reset (not reservoir-windowed); only `ravg` and the
   `r*` quantiles are reservoir-sampled estimates.
3. `mon.Event(name)` (a monkit `Meter`) emits `field="total"` (cumulative,
   rate()-safe) and `field="rate"`.
4. `environment.Register` (wired into every push via
   `pkg/telemetry.NewClient`) does **not** prefix measurements with
   `environment_`. Real measurement names: `goroutines`, `runtime_memstats`,
   `runtime_gcstats`, `rusage` (unix), `proc_stat`/`proc_statm` (linux),
   `fds` (unix), `process`. Confirmed via `pkg/telemetry/client_test.go`
   line ~652 (`key.Measurement == "goroutines"`).
5. `mon.Task()` auto-instrumentation emits `function` (fields: current,
   highwater, successes, errors, panics, failures, total; tagged `name`)
   and `function_times` (a duration distribution, tagged `name` + `kind`
   ("success"/"failure"), same field set as IntVal/DurationVal above) for
   every instrumented function - this is how gRPC/DB-layer rate+latency
   panels work with zero new instrumentation (DB layer = same mechanism,
   filtered to `coord/db.*` function names - no dedicated DB pool gauge
   exists in this codebase, grepped for sql.DBStats/pgxpool.Stat/
   StatSourceFromStruct and found none).

**Reputation**: only metric is `mon.Event("reputation_worker_disqualified")`
at `coord/reputation/service.go:76`. No suspend metric exists (suspension
is DB-only state, `Info.UnknownAuditSuspended`/`OfflineSuspended`).

**Ranged-loop** (pre-existing, not added by this branch's Tasks 3-10):
`ranged_loop_stats` (custom StatSource, standing gauge of last completed
pass: total/inline/remote objects/segments/bytes),
`ranged_loop_total_objects` etc. (IntVal one-shot-per-pass duplicates of
the same numbers), `ranged_loop_segments_processed` (IntVal, live-count),
`ranged_loop_suspicious_segments_count` (mon.Event, live-count
cross-check failure), `rangedloop_observer_duration` (DurationVal, tagged
`observer`, `-1` sentinel = errored not slow) - all in
`coord/rangedloop/{stats.go,livecount.go,service.go}`.

Full task report: `.superpowers/sdd/task-11-report.md` in the depin repo
worktree this ran in.
