---
type: design
project: aioz-stream
status: approved
created: 2026-09-09
repo: /home/tuan/work/stream/aioz-stream
worktree: /home/tuan/.treehouse/aioz-stream-370b08/2/aioz-stream
branch: refactor/sync-with-template-source
tags: [aioz-stream, aioz-common, observability, monkit, metrics, prometheus, debug-server]
---

# Design: replace the prometheus metrics package with aioz-common/stats

Continues the work in [[2026-09-07-align-aioz-stream-with-aioz-template]]. That
change adopted `aioz-common` for errors, uuid, lifecycle and the version stamp.
This one finishes the observability half: the hand-rolled prometheus metrics
package and the hand-rolled pprof server are replaced by `aioz-common/stats`,
and the Grafana/Loki stack leaves the repo.

All work lands in **worktree 2** (`refactor/sync-with-template-source`), not the
`feat/new-cdn` checkout.

## Starting state

| Thing | Now |
| --- | --- |
| `internal/utils/metrics` | prometheus `promauto.NewHistogramVec` x3: `db_latency`, `cache_latency`, `api_latency`, each labelled `query` |
| DB call sites | 345 `DbSum` observations across 35 files, all in `pkg/v1/repositories` |
| API call sites | 62 `ApiSum` observations across 14 `internal/app/<slice>/endpoint.go` files, each slice carrying its own identical `observe(operation string) func()` helper |
| `/metrics` | `promhttp.Handler()` mounted on the **public** echo server |
| pprof | hand-rolled `http.Server` on `127.0.0.1:6060`, started with a bare `go func()` in `cmd/http/main.go` |
| `/version` | echo route returning `aioz-common/version.Build` |
| monkit | not used anywhere |
| Monitoring stack | prometheus + grafana + promtail + node-exporter services in `docker-compose.yml`, config under `deploy/observability/` |

Two properties of the current setup motivate the change. The histograms record
latency and nothing else: a call that fails is indistinguishable from one that
succeeds, so error rate is unobservable. And `/metrics` is unauthenticated on
the public API port, exposing every internal query name and its latency profile.

## Decisions taken

1. **Full `mon.Task()` conversion**, not a monkit-backed shim. The point of the
   migration is the success/error/panic counts that come free with a task; a
   shim preserving the `Observe(seconds)` shape would keep the blind spot.
2. **Everything moves to the debug port.** Neither `/metrics` nor `/version`
   stays on the public echo server.
3. **`cmd/http` only.** `cmd/grpc` has no observability today and no lifecycle
   group; giving it one is a separate change. `aioz-stream-core` is a different
   module that has not adopted `aioz-common` at all.
4. **Series names break, and the monitoring stack leaves the repo.** Monitoring
   will run on a separate machine; how metrics are collected is decided later.
   No dashboards ship here. `/metrics` on the debug port is the whole contract.

## 1. `internal/app/debug`

A new package beside `internal/app/server`, with the same shape: it owns one
transport and knows nothing about any feature.

```go
// internal/app/debug/debug.go

// Server is the observability transport: monkit metrics, pprof, the build
// stamp, health and the dynamic log level, on a listener of its own.
type Server struct {
    log *slog.Logger
    srv *commondebug.Server
}

// New binds the listener eagerly, so a port clash is a startup error rather
// than a process that starts and silently has no observability.
func New(log *slog.Logger, cfg Config, level *slog.LevelVar) (*Server, error)

func (s *Server) Name() string                  { return "debug" }
func (s *Server) Run(ctx context.Context) error { return s.srv.Run(ctx) }
func (s *Server) Close() error                  { return s.srv.Close() }
```

`New` performs the two registrations that are otherwise silently forgotten:

```go
monitor.Register(monkit.Default)  // process/runtime series; without this
                                  // /metrics is empty until the first
                                  // instrumented call, so an idle service
                                  // looks identical to a dead one
version.Register(nil)             // build stamp as a monkit series
```

`Name/Run/Close` is exactly the existing `app.Chore` interface, so the server
drops into the `chores` slice in `cmd/http/main.go` with no new plumbing, and
becomes a `Peer.Debug` field when `app.Peer` is adopted. Nothing constructs
`app.Peer` yet - `cmd/http/main.go` still builds its lifecycle group inline.

It is registered **first**, so it is closed **last** and stays up through the
shutdown of everything else.

### Config

```go
type Config struct {
    Addr string `mapstructure:"addr" env:"DEBUG_ADDR"`
}
```

A local adapter is needed because `aioz-common`'s `debug.Config` is tagged
`help:` / `default:` (cfgstruct style) and aioz-stream loads config through
viper, which cannot bind those. Default `127.0.0.1:6060`, the same loopback
posture the pprof server has today.

### Removed from `cmd/http/main.go`

- the `pprofServer` goroutine and its `http.Server` literal
- the `_ "net/http/pprof"` blank import and its `//nolint:gosec` comment
- the `/metrics` promhttp route
- the `/version` echo route

### Endpoints after

All on `127.0.0.1:6060`, none public:
`/metrics` `/version/` `/health` `/top` `/mon/` `/logging` `/debug/pprof/*`

## 2. The 345 `DbSum` sites

One `var mon = monkit.Package()` in a new `pkg/v1/repositories/monkit.go`,
shared by all 35 files in the package. Then each method:

```go
func (r *apiKeyRepository) CreateApiKey(
    ctx context.Context, newApiKey *domain.ApiKey,
) (_ *domain.ApiKey, err error) {
    defer mon.Task()(&ctx)(&err)

    if err := r.db.WithContext(ctx).Create(newApiKey).Error; err != nil {
        return nil, err
    }

    return newApiKey, nil
}
```

Three mechanical edits per method: name the results, delete the
`t := time.Now()` / `defer func(){...}()` block, insert the task line. The
label string disappears - monkit reads the function name off the stack, and
every existing label already equals the method name, so no information is lost.
`time` and the `metrics` import fall out of most files; the compiler finds every
one.

### Traps

- **Shadowed `err`.** A method with a named `err` return whose body uses
  `if err := ...; err != nil { return nil, err }` never assigns the outer `err`,
  so monkit records a success on every failed call. Those bodies convert to
  `err = ...` followed by `if err != nil`. This is the one place where the
  conversion is not purely mechanical and where a careless pass produces
  metrics that are quietly wrong.
- **Bare `error` returns** need `(err error)`, not `(_ error)`.
- **Methods with no `ctx`** cannot take `mon.Task()(&ctx)`. Any such method
  either gains a context from its caller or uses `mon.Task()(&ctx)` against a
  local `ctx := context.Background()`; prefer threading the real context.

## 3. The 62 `ApiSum` sites

These are not converted one by one. Each of the 14 slices carries an identical
copy of:

```go
func observe(operation string) func() {
    start := time.Now().UTC()
    return func() {
        metrics.DbMetricsIns.ApiSum.WithLabelValues(operation).
            Observe(time.Since(start).Seconds())
    }
}
```

used as `defer observe("getUsage")()`. All of it is replaced by a single echo
middleware in `internal/app/server`:

```go
// monitoring instruments every route once, by the registered path and the
// status it answered with. Handlers say nothing about metrics at all.
func monitoring() echo.MiddlewareFunc
```

emitting a monkit duration and count keyed
`server_request{path="/api/usage/:id",method="GET",status="200"}`.

This deletes 62 defers, 14 helper functions and the last `metrics` imports
outside the repositories, and it covers the routes nobody remembered to
instrument - which today is most of them.

The trade is that the series key becomes the route template rather than the
hand-written operation id. Since the dashboards are being rebuilt anyway, the
route template is the better key: it cannot drift from the actual route.

## 4. Removals

| Removed | Why |
| --- | --- |
| `docker-compose.yml`: `prometheus`, `grafana`, `promtail`, `node-exporter` services | the stack runs on a separate machine now |
| `deploy/observability/` in full (`prometheus.yml`, `promtail-config.yaml`, `grafana/datasources.yaml`) | same |
| `internal/utils/metrics/` | replaced by monkit |
| `internal/utils/log/client.go` (`NewLokiClient`, `LokiClient`) | already dead code - nothing in the module calls it; removing it drops the `github.com/ic2hrmk/promtail` dependency |
| `github.com/prometheus/client_golang` from `go.mod` | last consumer gone |
| the `./libgrafana` volume mount, the promtail line in `.golangci.yml` | leftovers |

**Kept:** Graylog (`internal/utils/log/graylog.go`, `ObservabilityConfig.Graylog*`).
It is the live log sink, wired in both `cmd/http/init.go` and `cmd/grpc/init.go`,
and it is not part of the Grafana/Loki stack.

**Not added:** no Grafana dashboards, no scrape config, no alert rules in this
repo.

## Verification

- `go build ./...` and `go vet ./...` clean.
- `go test -race ./...` green.
- `golangci-lint run ./...` no new findings against the branch baseline.
- `grep -rn "utils/metrics" --include='*.go' .` returns nothing.
- `grep -rn "prometheus" go.mod` returns nothing.
- Boot `cmd/http`, then assert against `127.0.0.1:6060`:
  - `/metrics` is non-empty **before any request is served** (proves
    `monitor.Register` ran),
  - `/version/` returns the build stamp,
  - `/health` returns OK,
  - `/debug/pprof/goroutine?debug=1` returns a dump.
- Assert the public API port serves **404** for `/metrics` and `/version`.
- Drive one repository call that fails and confirm `error_count` increments on
  its series - this is the check that catches the shadowed-`err` trap, and it
  must be run against a method that was converted by hand rather than one that
  happened to be trivial.

## Follow-ups, not in scope

- `cmd/grpc` observability, which needs a lifecycle group first.
- `aioz-stream-core`, which has not adopted `aioz-common`.
- Whatever collects `/metrics` from the separate monitoring machine.
- The `aioz-common` improvements listed in [[aioz-common-shared-package-gaps]].
