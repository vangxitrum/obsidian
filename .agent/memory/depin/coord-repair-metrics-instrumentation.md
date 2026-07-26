---
type: decision
tags: [depin, telemetry, monkit, repair, testing]
created: 2026-07-09
agent: subagent (Task 4 executor)
---

Task 4 of the "coordinator metrics collection" plan (branch
`feat/coord-metrics-collection`, commit `c138c7e`
"feat(coord/repair): add monkit instrumentation to checker/queue/repairer
pipeline"). Independent of [[telemetry-remote-write-push]] and
[[coord-audit-metrics-instrumentation]] (Tasks 1-3); same repo, same monkit
patterns.

## What was added
- **Checker** (`coord/repair/checker/observer.go`, `observerFork.process`):
  `repair_segments_healthy` / `repair_segments_injured` /
  `repair_segments_over_threshold` (a 3-way partition I derived, since the code
  doesn't natively bucket into these 4 named states - see judgment calls below) +
  orthogonal `repair_segments_clumped`; `repair_segments_enqueued` wired into the
  pre-existing `newInsertCallback func()` param of `InsertBuffer.Insert` (was a
  no-op `func(){}` before - literally the pre-built "for metrics" hook per that
  method's own doc comment).
- **Queue** (`coord/repair/queue/insertbuffer.go`, `InsertBuffer.Flush`) and
  **repairer** (`coord/repair/repairer/repairer.go`, `Worker.Run`, after
  `Select`): `repair_queue_length` gauge via the queue's own pre-existing
  `Count(ctx)` interface method (had no other caller in the codebase before this -
  purpose-built for cheap reporting, is how "select and insert" both update the
  gauge without a bespoke new query).
- **Repairer outcome** (`coord/repair/repairer/segments.go`,
  `SegmentRepairer.Repair` + `repairReedSolomon`/`repairClone`):
  `repair_strategy_rs`/`repair_strategy_clone` on dispatch (regardless of
  outcome); `repair_duration` measured strictly around the RS/CLONE call;
  `repair_failed` once in `Repair()` on `err != nil` (not scattered across the
  ~9 individual `return false, err` sites per strategy function);
  `repair_success` + `repair_pieces_reconstructed` (`IntVal.Observe(len(added))`,
  reusing the exact value already logged via `zap.Int("added", ...)`) inside each
  strategy function at its true terminal success point (after
  `UpdateSegmentPieces`).

## Gotcha: unlike Task 3's finding, no monkit.go dance needed here
All three sub-packages (`checker`, `queue`, `repairer`) already had their own
`var mon = monkit.Package()` before this task - confirmed via `grep -rn
"monkit.Package()" coord/repair/` before writing anything. So "one per
sub-package" was already the established layout; just reused what existed. No
`monkit.go` files created.

## File-placement deviation from the brief, flagged in the report
The plan's brief named `coord/repair/repairer/repairer.go` as the file for
success/failed/strategy/pieces-reconstructed/duration. The real decision points
(strategy algorithm, piece counts) live in `segments.go` (`SegmentRepairer.Repair`
and its two strategy methods) - `repairer.go`'s `Worker` only sees `(bool,
error)` from `Repair()`. Same package (`repairer`), so still within the "one
per sub-package" boundary; only used `repairer.go` for the thing it actually
controls (the queue-length gauge on `Select`). Self-review checklist phrasing
("...in repairer") matches the sub-package, not the literal filename - lean on
that phrasing over the brief's file list when they conflict, after confirming via
"find the real site" instruction.

## Test pattern: reused Task 3's `withTestMon` isolation exactly
Same swap-package-`mon`-var-to-a-scoped-registry pattern from
[[coord-audit-metrics-instrumentation]], one `monkit_test.go` per sub-package
(`checker`, `queue`, `repairer`). New this time: `intValLast(scope, name)` reads
back an `IntVal`'s **`"recent"`** field (not `"last"` - that field name doesn't
exist; `IntDist.Stats` emits `count/sum/min/max/rmin/ravg/r10/r50/r90/r99/rmax/
recent`) for gauge/reconstructed-pieces assertions.

For the repairer's `Worker.Run` gauge test (`repairer_test.go`), avoided a
sleep-based race by having the fake queue's `Count()` send its result over an
**unbuffered** channel (blocks until the test receives), then the test calls
`cancel()` and blocks on `<-done` (the `Run()` goroutine's exit) before reading
the gauge - joining the goroutine gives a happens-before guarantee that the
`Observe()` call (which runs immediately after `Count()` returns, same
goroutine, no further blocking ops before the post-cancel `sleep()`) has already
executed. General pattern for testing monkit side-effects inside a goroutine-run
polling loop without `time.Sleep` flakiness.

For the repairer's real-outcome tests (`segments_test.go`), built full fakes for
`OrderService`/`Overlay`/`downloader`/`Uploader`/`SegmentStore` and ran a genuine
CLONE repair + a genuine k=2/n=4 Reed-Solomon repair (real `ers_schema`/
`infectious` encode+reconstruct, same library `coord/repair/repairer/ec_test.go`
already exercises) rather than stubbing the reconstruction math - proves the
counter fires on the actual success branch, not a mocked shortcut.

## Judgment calls flagged in the task report (not blocking, but worth knowing)
1. `repair_segments_healthy`/`_injured`/`_over_threshold` is my own 3-way
   partition (healthy = zero unhealthy pieces; injured = the branch that reaches
   the queue insert; over_threshold = has unhealthy pieces but doesn't trigger
   repair yet) - the checker code itself only exposes `piecesCheck`
   (Missing/Suspended/Clumped/Exiting/OutOfPlacement/InExcludedCountry/Unhealthy/
   Healthy sets) and two booleans, not these 4 named buckets.
2. `repair_success + repair_failed` does not sum to total `Repair()` calls - the
   several "return true, nil" no-op/skip branches (segment gone, no root piece,
   unsupported algo, already-healthy-by-now, irreparable/give-up) increment
   neither counter.
3. `repair_queue_length` costs one extra `Count()` DB query per select-batch and
   per insert-flush - judged as "using an existing purpose-built cheap method"
   rather than "adding a new query," per the brief's "cheaply available" wording,
   but it's a literal extra query nonetheless.
