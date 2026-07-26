---
type: decision
tags: [depin, telemetry, monkit, prometheus, protobuf]
created: 2026-07-09
agent: subagent (Task 1 + Task 2 executors)
---

Implemented `pkg/telemetry` in aioz-depin/depin: a Prometheus remote_write
push client (Task 1, branch `feat/coord-metrics-collection`, commit
`5ca6ea6`), then wired it into every coord role (Task 2, same branch,
commit `c8c973d`). Multi-task plan lives at `.superpowers/sdd/task-{1,2}-brief.md`
/ `task-{1,2}-report.md` in the repo.

## What it is
- `Client{Run(ctx) error, Close() error}` matches `lifecycle.Item{Run, Close}`
  exactly (same shape as `coord/peer.go`'s debug-server registration). Built
  on `sync2.Cycle`, the same periodic-loop primitive used by other `coord/*`
  chores (`expireddeletion.Chore`, `rangedloop.Service`, `gc/sender.Service`)
  - not a hand-rolled ticker.
- Walks a `monkit.Registry` and POSTs snappy-compressed remote_write
  `TimeSeries` to `Config.URL` (empty = disabled, off by default). Critically
  **does not** apply `monkit.NewDeltaTransformer()` like the pull path
  (`pkg/debug/prometheus.go`) does - reports raw cumulative values, since
  remote_write receivers compute `rate()`/`increase()` themselves.
- `Config` has a cardinality filter: `IncludePrefixes`/`ExcludePrefixes` on
  measurement name, `DropQuantiles` (drops monkit's `r10/r50/r90/r99/rmin/
  rmax` reservoir fields, keeps `count/sum/min/max`).

## Protobuf dependency decision
Rejected `github.com/castai/promwrite` (the obvious lightweight remote-write
client) after inspecting its `go.mod`: it imports
`github.com/prometheus/prometheus/prompb` directly, which in turn imports
that module's internal `model/labels`, `model/exemplar`, `model/histogram`
packages - i.e. it is not actually a small dependency, it just repackages a
slice of the full `prometheus/prometheus` module. That's exactly the weight
the task wanted avoided.

**How to apply**: instead, hand-authored a minimal proto with only the 4
needed messages (`WriteRequest`/`TimeSeries`/`Label`/`Sample`), field numbers
copied exactly from upstream `prompb/{remote,types}.proto` for wire
compatibility, generated via this repo's existing `buf generate` +
`protoc-gen-gogofaster` toolchain: `buf generate --path
pkg/pb/telemetry/remote/v1/remote.proto`. Lives at
`pkg/pb/telemetry/remote/v1/`, follows the same `go_package`/directory
convention as `pkg/pb/worker/piece/v1/`. Zero new third-party deps added;
only promoted the already-indirect `github.com/golang/snappy` to direct in
`go.mod` (single-line change - avoid running a blanket `go mod tidy` here,
it also wants to drop an unrelated pre-existing unused dep,
`github.com/inconshreveable/go-update`, which is out of scope for a
telemetry-only commit).

## Gotcha found via TDD: monkit environment.Register guard must be per-registry
`spacemonkeygo/monkit/v3/environment.Register(registry)` is **not**
idempotent (it re-chains its stat sources every call), so double-registering
double-counts CPU/mem/goroutine/GC/fd stats. The instinct is to guard it with
a single package-level `sync.Once` - **don't**. That's wrong the moment more
than one `*monkit.Registry` exists in the process: the first `NewClient`
call (against whatever registry it happens to get) permanently trips the
Once, so every subsequent call against a *different* registry silently
registers nothing. Production only ever uses one registry
(`monkit.Default`) so this wouldn't bite there, but it bit test isolation
immediately (multiple tests each construct their own fresh registry) and is
a latent trap for any future multi-registry use of this pattern anywhere
else in the codebase. Fix: guard with a mutex + `map[*monkit.Registry]bool`,
keyed per registry, not a bare `sync.Once`.

## Task 2: wired into every coord role (done)
- Added `Metrics telemetry.Config` as a top-level field on `coord.Config`
  (`coord/config.go`, next to `Debug debug.Config`) and on `relay.Config`
  (`coord/relay/config.go`) - not nested under `AppConfig`, matching the
  Debug field's flat style.
- Added `func (b *base) setupTelemetry()` in `coord/peer.go`, called from
  the *end* of the existing `setupDebug()`. Since all 6 coord roles (`peer`
  = combined `run`, `api`, `core`, `audit`, `repair`, `rangeloop`) embed
  `*base` and already call `base.setupDebug()`, this one call site wires
  telemetry into every role without touching each role file individually.
- `coord/relay/peer.go` (`relay.Peer`) is **not** built on `base` and has
  **no separate `setupDebug()` method at all** - its debug-server setup is
  inlined directly in `NewPeer`. Added the equivalent telemetry block
  inline there too, right after the inline debug block. Don't assume every
  role file has a `setupDebug()` to hook into - grep first.
- **Off-by-default guard**: `if b.Config.Metrics.URL == "" { return }`
  happens *before* `telemetry.NewClient` is ever called - not just before
  the `lifecycle.Item` registration. This matters because `NewClient`
  unconditionally calls `environment.Register(registry)` against whatever
  registry it's given (guarded per-registry, see above, but still adds new
  process/runtime series the *first* time). Calling `NewClient`
  unconditionally would silently change `monkit.Default`'s pull-path
  `/metrics` output even with telemetry push fully disabled. Passed
  `monkit.Default` as the registry in both `base.setupTelemetry()` and
  relay's inline wiring - same one `pkg/debug`'s `debug.NewServer` already
  reads (`coord/peer.go` `debug.NewServer(..., monkit.Default, ...)`), so
  push and pull paths report on the same data.
- **Env var naming confirmed two ways**: (1) built the binary, ran
  `bin/coord {api,run,relay} --help | grep -i metrics` → flags are
  `--metrics.<field>` (e.g. `--metrics.url`, `--metrics.bearer-token`); (2)
  proved it end-to-end by running
  `AIOZ_METRICS_URL=... AIOZ_METRICS_ROLE=... bin/coord setup --config-dir <tmp>`
  and inspecting the generated `config.yaml`, which showed the env-provided
  values resolved into `metrics.url:` / `metrics.role:` - stronger evidence
  than `--help` alone that viper's `SetEnvPrefix("aioz")` +
  `SetEnvKeyReplacer(".", "_", "-", "_")` (`pkg/process/exec_conf.go`)
  actually binds nested struct fields, since this repo has a known gotcha
  around nested-struct-tag env prefixes.
- **Reusable test pattern**: `lifecycle.Group` (`pkg/lifecycle/group.go`)
  has no public way to introspect which items were registered - `items`
  is unexported and there's no `Len()`/`Items()` accessor. To test
  "off-by-default registers no lifecycle item" without reaching into
  private state via reflection, capture the group's own `"started"` debug
  log line (`group.log.Debug("started", zap.Strings("items", started))`)
  via `zaptest/observer.New(zapcore.DebugLevel)` and read the `"items"`
  field back out with `LoggedEntry.ContextMap()["items"].([]interface{})`.
  For the enabled case, just exercise `Group.Run()` end-to-end against a
  real `httptest.Server` and assert on the actual HTTP request received -
  more reliable than trying to assert on internal wiring state. See
  `coord/telemetry_test.go` (`TestSetupTelemetry_DisabledByDefault`,
  `TestSetupTelemetry_EnabledPushesWithinInterval`) and
  `coord/relay/peer_test.go` (`TestNewPeer_TelemetryPushesWithinInterval`).

See also [[worker-autoupdate]] for another `coord/*` daemon following the
same `sync2.Cycle` + `lifecycle.Item` wiring pattern, and
[[coord-audit-metrics-instrumentation]] for Task 3 of this same plan (domain
`monkit` counters in `coord/audit`, independent of the push client itself).
