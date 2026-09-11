---
type: plan
project: aioz-stream
status: ready
created: 2026-09-09
spec: "[[2026-09-09-monkit-observability-migration]]"
worktree: /home/tuan/.treehouse/aioz-stream-370b08/2/aioz-stream
branch: refactor/sync-with-template-source
tags: [aioz-stream, aioz-common, observability, monkit, metrics, plan]
---

# Monkit Observability Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace aioz-stream's hand-rolled prometheus metrics package and pprof
server with `aioz-common/stats` monkit instrumentation behind a single debug
server, and remove the Grafana/Loki stack from the repository.

**Architecture:** A new `internal/app/debug` package wraps
`aioz-common/stats/debug.Server` and satisfies the existing `app.Chore`
interface, so it registers in the lifecycle group beside every other component.
Repository methods are instrumented with `defer mon.Task()(&ctx)(&err)`; HTTP
handlers are instrumented once by an echo middleware in `internal/app/server`
rather than 62 times by hand. `/metrics` and `/version` move off the public API
port onto the debug listener.

**Tech Stack:** Go 1.25, `github.com/spacemonkeygo/monkit/v3 v3.0.24`,
`gitlab.internal/tuan.quang.tran/aioz-common v0.2.1-0.20260908094255-93c696c1e41d`,
`github.com/labstack/echo/v4`, `gorm.io/gorm v1.25.12`.

**Spec:** `Projects/aioz-stream/plans/2026-09-09-monkit-observability-migration.md`
([[2026-09-09-monkit-observability-migration]])

## Global Constraints

- **All work happens in worktree 2:** `/home/tuan/.treehouse/aioz-stream-370b08/2/aioz-stream`,
  branch `refactor/sync-with-template-source`. Do **not** touch
  `/home/tuan/work/stream/aioz-stream`, which is on `feat/new-cdn` and has
  unrelated uncommitted changes.
- **Build tag `purego` is required.** Every `go build` / `go test` invocation
  uses `-tags purego`. The Makefile sets `BUILD_TAGS ?= purego`; prefer
  `make test` over a bare `go test`.
- Tests run with `-race`: `make test` is `go test -race -tags purego ./...`.
- Module path prefix is `10.0.0.50/tuan.quang.tran/vms-v2`.
- Import grouping is enforced by `goimports -local`. Follow the existing
  three-block order in each file: stdlib, third-party, then
  `gitlab.internal/tuan.quang.tran/...`, then `10.0.0.50/tuan.quang.tran/vms-v2/...`.
- **The global Go formatter hook re-wraps whole files on `Edit`.** For large
  files use Bash (`sed`, `python`) edits instead, or the diff explodes with
  unrelated reflowing. This is a documented recurring trap on this repo.
- Never key a monkit series on rendered SQL or on a user-supplied string;
  cardinality must stay bounded.
- Commit after every task. Conventional Commits format, no `Co-Authored-By`
  trailer, no em-dashes in the message.

## File Structure

**Created**

| File | Responsibility |
| --- | --- |
| `internal/app/debug/debug.go` | The observability transport: wraps `aioz-common/stats/debug.Server`, performs the monitor/version registrations, exposes `Name/Run/Close` |
| `internal/app/debug/config.go` | Viper-shaped `Config` (`mapstructure`/`env` tags) plus `DefaultConfig` and `Validate` |
| `internal/app/debug/debug_test.go` | Asserts the endpoints answer and that `/metrics` is non-empty before any application call |
| `internal/app/server/monitoring.go` | The echo middleware that instruments every route once |
| `internal/app/server/monitoring_test.go` | Asserts the middleware emits a series keyed by route template, method and status |
| `pkg/v1/repositories/monkit.go` | The package-scoped `var mon = monkit.Package()` shared by all 35 repository files |
| `pkg/v1/repositories/monkit_test.go` | Test helper that swaps `mon` to a scoped registry, plus the shadowed-`err` regression test |

**Modified**

| File | Change |
| --- | --- |
| `cmd/http/main.go` | Remove the pprof goroutine, the `net/http/pprof` blank import, the `/metrics` and `/version` routes; construct and register the debug server |
| `internal/app/server/server.go` | Insert the monitoring middleware into the chain |
| `internal/config/config.go` | Add `Debug DebugConfig` to `AppConfig` |
| `pkg/v1/repositories/*.go` (35 files) | `mon.Task()` conversion, 345 sites |
| `internal/app/*/endpoint.go` and `internal/app/media/*_endpoint.go` (14 files) | Delete the `observe` helpers and their 62 call sites |
| `docker-compose.yml` | Remove the `prometheus`, `grafana`, `promtail`, `node-exporter` services and the now-unused `monitor` network |
| `go.mod` / `go.sum` | Drop `github.com/prometheus/client_golang` and `github.com/ic2hrmk/promtail` |
| `env-example/app.env` | Add `DEBUG_ADDR`, drop the Grafana comment |
| `.golangci.yml` | Drop the promtail line from the logging exclusion comment |

**Deleted**

- `internal/utils/metrics/` (whole package)
- `internal/utils/log/client.go` (dead Loki client)
- `deploy/observability/` (whole directory)

---

### Task 1: The debug transport

**Files:**
- Create: `internal/app/debug/config.go`
- Create: `internal/app/debug/debug.go`
- Test: `internal/app/debug/debug_test.go`

**Interfaces:**
- Consumes: `gitlab.internal/tuan.quang.tran/aioz-common/stats/debug` (aliased
  `commondebug`), `.../stats/monitor`, `.../version`.
- Produces:
  - `debug.Config` with field `Addr string`
  - `func debug.DefaultConfig() Config`
  - `func (Config) Validate() error`
  - `func debug.New(log *slog.Logger, cfg Config, level *slog.LevelVar) (*Server, error)`
  - `func (*Server) Name() string`, `func (*Server) Run(ctx context.Context) error`,
    `func (*Server) Close() error`, `func (*Server) Addr() string`

Task 2 registers this as an `app.Chore`; `Name/Run/Close` is exactly that
interface. `Addr()` exists so the test can bind port 0 and still find the
server.

- [ ] **Step 1: Write the failing test**

Create `internal/app/debug/debug_test.go`:

```go
package debug_test

import (
	"context"
	"io"
	"log/slog"
	"net/http"
	"strings"
	"testing"
	"time"

	"10.0.0.50/tuan.quang.tran/vms-v2/internal/app/debug"
)

// serve starts a debug server on a random loopback port and returns its base
// URL. Port 0, so the test never collides with a real listener or with itself.
func serve(t *testing.T) string {
	t.Helper()

	ctx, cancel := context.WithCancel(context.Background())

	cfg := debug.DefaultConfig()
	cfg.Addr = "127.0.0.1:0"

	srv, err := debug.New(slog.New(slog.DiscardHandler), cfg, nil)
	if err != nil {
		cancel()
		t.Fatalf("debug.New: %v", err)
	}

	done := make(chan error, 1)
	go func() { done <- srv.Run(ctx) }()

	t.Cleanup(func() {
		cancel()

		if err := srv.Close(); err != nil {
			t.Errorf("Close: %v", err)
		}

		select {
		case <-done:
		case <-time.After(5 * time.Second):
			t.Error("debug server did not stop")
		}
	})

	return "http://" + srv.Addr()
}

func get(t *testing.T, url string) (int, string) {
	t.Helper()

	req, err := http.NewRequestWithContext(
		context.Background(), http.MethodGet, url, nil,
	)
	if err != nil {
		t.Fatalf("NewRequest %s: %v", url, err)
	}

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		t.Fatalf("GET %s: %v", url, err)
	}
	defer func() { _ = resp.Body.Close() }()

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		t.Fatalf("read %s: %v", url, err)
	}

	return resp.StatusCode, string(body)
}

// TestEndpoints is the contract the deployment depends on: these paths exist
// on the debug listener and nowhere else.
func TestEndpoints(t *testing.T) {
	base := serve(t)

	for _, path := range []string{
		"/metrics",
		"/version/",
		"/health",
		"/top",
		"/debug/pprof/goroutine?debug=1",
	} {
		status, body := get(t, base+path)
		if status != http.StatusOK {
			t.Errorf("GET %s: status = %d, want 200", path, status)
		}

		if body == "" {
			t.Errorf("GET %s: empty body", path)
		}
	}
}

// TestMetricsPopulatedBeforeAnyCall is the reason New calls monitor.Register.
// Without it monkit knows only the series some package has already
// incremented, so a freshly started service serves an empty /metrics and an
// idle service is indistinguishable from a dead one.
func TestMetricsPopulatedBeforeAnyCall(t *testing.T) {
	base := serve(t)

	_, body := get(t, base+"/metrics")

	if !strings.Contains(body, "goroutines") {
		t.Errorf("/metrics has no process series; monitor.Register did not run.\nbody:\n%s", body)
	}
}

// TestVersionSeriesRegistered proves version.Register ran, so a dashboard can
// tell which build produced a sample.
func TestVersionSeriesRegistered(t *testing.T) {
	base := serve(t)

	_, body := get(t, base+"/metrics")

	if !strings.Contains(body, "version") {
		t.Errorf("/metrics carries no version series; version.Register did not run.\nbody:\n%s", body)
	}
}

// TestValidateRejectsEmptyAddr keeps a misconfiguration a startup error rather
// than a server that binds somewhere unintended.
func TestValidateRejectsEmptyAddr(t *testing.T) {
	cfg := debug.DefaultConfig()
	cfg.Addr = ""

	if err := cfg.Validate(); err == nil {
		t.Error("Validate() = nil, want an error for an empty address")
	}
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd /home/tuan/.treehouse/aioz-stream-370b08/2/aioz-stream && go test -tags purego ./internal/app/debug/...`

Expected: FAIL — the package `internal/app/debug` does not exist, so the build
fails with `no required module provides package`.

- [ ] **Step 3: Write the config**

Create `internal/app/debug/config.go`:

```go
package debug

import "github.com/zeebo/errs/v2"

// Error tags every failure that originates in the debug transport.
var Error = errs.Tag("debug")

// Config configures the debug listener.
//
// aioz-common's debug.Config is tagged for aioz-config (help: / default:),
// which viper cannot bind. This is the viper-shaped half; New maps it across.
// When aioz-stream adopts aioz-common/config, this type goes away.
type Config struct {
	Addr string `mapstructure:"addr" env:"DEBUG_ADDR"`
}

// DefaultConfig binds to loopback. /metrics names every internal operation and
// /debug/pprof serves heap dumps and 30-second CPU profiles, so a public
// listener is both an information leak and a way to stall the process. A
// deployment that scrapes from another host overrides DEBUG_ADDR and takes
// responsibility for the firewall rule.
func DefaultConfig() Config {
	return Config{
		Addr: "127.0.0.1:6060",
	}
}

// Validate reports settings that parse but cannot work.
func (c Config) Validate() error {
	if c.Addr == "" {
		return Error.Errorf("debug.addr must not be empty")
	}

	return nil
}
```

- [ ] **Step 4: Write the server**

Create `internal/app/debug/debug.go`:

```go
// Package debug is the observability transport. It owns the debug listener and
// knows nothing about any feature: monkit metrics, pprof, the build stamp,
// health and the dynamic log level, on a port of its own.
package debug

import (
	"context"
	"log/slog"
	"net"

	"github.com/spacemonkeygo/monkit/v3"

	commondebug "gitlab.internal/tuan.quang.tran/aioz-common/stats/debug"
	"gitlab.internal/tuan.quang.tran/aioz-common/stats/monitor"
	"gitlab.internal/tuan.quang.tran/aioz-common/version"
)

// Server serves the observability endpoints.
type Server struct {
	log      *slog.Logger
	srv      *commondebug.Server
	listener net.Listener
}

// New binds the listener and registers the process statistics.
//
// Binding here rather than in Run makes a port clash a startup error, which is
// a much cheaper failure than a process that starts, reports healthy and
// silently has no observability.
//
// level may be nil. When it is non-nil the /logging endpoint can change the
// root handler's level at runtime without a restart.
func New(log *slog.Logger, cfg Config, level *slog.LevelVar) (*Server, error) {
	if err := cfg.Validate(); err != nil {
		return nil, Error.Wrap(err)
	}

	listener, err := net.Listen("tcp", cfg.Addr)
	if err != nil {
		return nil, Error.Wrap(err)
	}

	// Both registrations are wanted in every deployment and neither is done by
	// NewServer. Forgetting them fails silently: without monitor.Register
	// monkit knows only the series some package has already incremented, so
	// /metrics is empty until the first instrumented call and an idle service
	// looks exactly like a dead one. monitor.Register is idempotent per
	// registry, so calling it here is safe even if a caller also does.
	monitor.Register(monkit.Default)
	version.Register(nil)

	srv := commondebug.NewServerWithLevel(
		log,
		listener,
		monkit.Default,
		commondebug.Config{Addr: cfg.Addr},
		level,
	)

	return &Server{log: log, srv: srv, listener: listener}, nil
}

// Name identifies the server in the lifecycle group and in logs.
func (s *Server) Name() string { return "debug" }

// Addr is the bound address, which is how a caller that asked for port 0
// learns the port it actually got.
func (s *Server) Addr() string { return s.listener.Addr().String() }

// Run serves until ctx is canceled.
func (s *Server) Run(ctx context.Context) error {
	return Error.Wrap(s.srv.Run(ctx))
}

// Close stops the server. It is safe to call more than once.
func (s *Server) Close() error {
	return Error.Wrap(s.srv.Close())
}
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `go test -race -tags purego ./internal/app/debug/...`
Expected: PASS, all five tests.

If `TestVersionSeriesRegistered` fails, check what measurement name
`version.Register` actually produces by printing the `/metrics` body, and
assert on that exact name rather than loosening the test.

- [ ] **Step 6: Commit**

```bash
git add internal/app/debug
git commit -m "feat(debug): add the observability transport

Wraps aioz-common/stats/debug behind the app.Chore shape, and performs the
monitor and version registrations that NewServer does not do and that fail
silently when forgotten."
```

---

### Task 2: Wire the debug server in, remove the hand-rolled one

**Files:**
- Modify: `internal/config/config.go` (add `Debug` to `AppConfig`, around line 47)
- Modify: `cmd/http/main.go` (imports at 10-26, pprof block at 63-76, routes at 78-86, chores at 228-247)

**Interfaces:**
- Consumes: `debug.New`, `debug.DefaultConfig`, `debug.Config` from Task 1.
- Produces: `config.AppConfig.Debug` of type `debug.Config`.

- [ ] **Step 1: Write the failing test**

Append to `internal/config/config_test.go`:

```go
// TestDebugAddrDefault pins the loopback default. A deployment that wants to
// be scraped from another host sets DEBUG_ADDR and owns the firewall rule;
// nothing should make the debug port public by accident.
func TestDebugAddrDefault(t *testing.T) {
	cfg := debugpkg.DefaultConfig()

	if cfg.Addr != "127.0.0.1:6060" {
		t.Errorf("Addr = %q, want 127.0.0.1:6060", cfg.Addr)
	}
}
```

Add the import `debugpkg "10.0.0.50/tuan.quang.tran/vms-v2/internal/app/debug"`
to that file's import block.

- [ ] **Step 2: Run it to verify it fails**

Run: `go test -tags purego ./internal/config/...`
Expected: FAIL — the import does not resolve until Task 1 is merged; if Task 1
is already merged this passes immediately, which is fine. The real verification
for this task is Step 6.

- [ ] **Step 3: Add the config field**

In `internal/config/config.go`, add the import:

```go
	debugpkg "10.0.0.50/tuan.quang.tran/vms-v2/internal/app/debug"
```

and add one field to `AppConfig`, after `Observability` (line 46):

```go
	Debug         debugpkg.Config     `mapstructure:"debug"`
```

Then, wherever `MustNewAppConfig` sets defaults before unmarshalling, seed the
default so an unset `debug` block still binds to loopback:

```go
	viper.SetDefault("debug.addr", debugpkg.DefaultConfig().Addr)
```

Read the surrounding lines of `MustNewAppConfig` first and match how the other
defaults are set; if the function uses a different mechanism than
`viper.SetDefault`, follow that one instead.

- [ ] **Step 4: Remove the hand-rolled pprof server and the two routes**

In `cmd/http/main.go` delete, in this order:

1. Lines 10-11, the blank import and its lint suppression:

```go
	//nolint:gosec // G108: /debug/pprof is served on a loopback-only listener, see below
	_ "net/http/pprof"
```

2. Line 20, the promhttp import:

```go
	"github.com/prometheus/client_golang/prometheus/promhttp"
```

3. Lines 63-76, the whole pprof goroutine:

```go
	// pprof. Bound to loopback: /debug/pprof serves heap dumps and 30-second CPU
	// profiles, and net/http has no default timeouts, so a public listener is
	// both an information leak and a way to stall the process.
	go func() {
		pprofServer := &http.Server{
			Addr:              "127.0.0.1:6060",
			ReadHeaderTimeout: 10 * time.Second,
		}
		if err := pprofServer.ListenAndServe(); err != nil &&
			!errors.Is(err, http.ErrServerClosed) {
			slog.Error("pprof server", "err", err)
		}
	}()
```

4. Lines 77-86, both routes:

```go
	// The build stamp, so a running deployment can say which commit it is.
	server.GET("/version", func(c echo.Context) error {
		return c.JSON(http.StatusOK, version.Build)
	})

	// metrics
	server.GET("/metrics", func(c echo.Context) error {
		promhttp.Handler().ServeHTTP(c.Response().Writer, c.Request())
		return nil
	})
```

5. The now-unused `"gitlab.internal/tuan.quang.tran/aioz-common/version"` import
   at line 26, **only if** nothing else in the file still references
   `version.`. Check with `grep -n 'version\.' cmd/http/main.go` first.

Use Bash edits (`sed -i` with line ranges, or `python`) rather than the Edit
tool: this file is long and the formatter hook reflows whole files on Edit.

- [ ] **Step 5: Register the debug server in the lifecycle group**

In `cmd/http/main.go`, immediately before line 244's
`services := lifecycle.NewGroup(...)`, construct the server:

```go
	// The observability transport. It is registered before every chore, so it
	// is closed after them: metrics and pprof stay reachable through the
	// shutdown of everything else, which is exactly when a hung component
	// needs to be diagnosed.
	debugServer, err := debugpkg.New(
		slog.Default().With("component", "debug"),
		appConfig.Debug,
		nil,
	)
	if err != nil {
		slog.Error("debug server", "err", err)
		os.Exit(1)
	}
```

and register it as the first item, immediately after the group is created and
**before** the `for _, chore := range chores` loop at line 245:

```go
	services.Add(lifecycle.Item{
		Name:  debugServer.Name(),
		Run:   debugServer.Run,
		Close: debugServer.Close,
	})
```

Add the import `debugpkg "10.0.0.50/tuan.quang.tran/vms-v2/internal/app/debug"`.

- [ ] **Step 6: Build and verify by hand**

Run: `make build` then start the binary against your local `debug.env`.

Then assert, from another shell:

```bash
curl -sf http://127.0.0.1:6060/metrics  | head -5    # non-empty
curl -sf http://127.0.0.1:6060/version/               # build stamp JSON
curl -sf http://127.0.0.1:6060/health                 # OK
curl -s  -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/metrics   # want 404
curl -s  -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/version   # want 404
```

Substitute the real API port from your `debug.env` for 8080. The two 404s are
the point of the task: they prove the endpoints left the public listener.

- [ ] **Step 7: Commit**

```bash
git add cmd/http/main.go internal/config/config.go internal/config/config_test.go
git commit -m "refactor(http): serve observability on the debug port

Replaces the hand-rolled pprof listener and moves /metrics and /version off
the public API. /metrics named every internal operation and its latency to
anyone who asked, and pprof served heap dumps from a server nothing could
shut down."
```

---

### Task 3: The HTTP monitoring middleware

**Files:**
- Create: `internal/app/server/monitoring.go`
- Create: `internal/app/server/monitoring_test.go`
- Modify: `internal/app/server/server.go:127` (the middleware chain)

**Interfaces:**
- Produces: `func monitoring() echo.MiddlewareFunc`, unexported, used only by
  `server.New`. Task 4 relies on the series it emits existing, so that the 62
  hand-written observations can be deleted without losing coverage.
- Series emitted: measurement `server_request`, tags `path` (the **route
  template**, e.g. `/api/v1/stub`), `method`, `status`.

- [ ] **Step 1: Write the failing test**

Create `internal/app/server/monitoring_test.go`:

```go
package server_test

import (
	"net/http"
	"testing"

	"github.com/spacemonkeygo/monkit/v3"
)

// findSeries reports whether monkit.Default carries a server_request series
// with these three tags. It reads the default registry because the middleware
// uses the package scope, which is what production scrapes.
func findSeries(t *testing.T, path, method, status string) bool {
	t.Helper()

	found := false

	monkit.Default.Stats(func(key monkit.SeriesKey, field string, val float64) {
		if key.Measurement != "server_request" {
			return
		}

		if key.Tags.Get("path") == path &&
			key.Tags.Get("method") == method &&
			key.Tags.Get("status") == status {
			found = true
		}
	})

	return found
}

// TestMonitoringKeysOnRouteTemplate is the property that makes one middleware
// able to replace 62 hand-written observations: the series key comes from the
// registered route, so it cannot drift from the route and it cannot be
// forgotten on a new handler.
func TestMonitoringKeysOnRouteTemplate(t *testing.T) {
	base := serve(t, stub{})

	status, _ := get(t, base+"/api/v1/stub")
	if status != http.StatusNotFound {
		t.Fatalf("stub route: status = %d, want 404", status)
	}

	if !findSeries(t, "/api/v1/stub", http.MethodGet, "404") {
		t.Error("no server_request series for the stub route")
	}
}

// TestMonitoringRecordsUnmatchedRoutes keeps an unrouted request from creating
// a series per URL. A 404 for a path echo never registered must not put the
// raw path into a tag, or a scanner walking the site explodes the cardinality.
func TestMonitoringRecordsUnmatchedRoutes(t *testing.T) {
	base := serve(t, stub{})

	if _, _ = get(t, base+"/no/such/route/12345"); false {
		return
	}

	monkit.Default.Stats(func(key monkit.SeriesKey, field string, val float64) {
		if key.Measurement != "server_request" {
			return
		}

		if key.Tags.Get("path") == "/no/such/route/12345" {
			t.Error("unmatched route put the raw path into a tag; cardinality is unbounded")
		}
	})
}
```

This reuses the `serve`, `get` and `stub` helpers already in
`internal/app/server/server_test.go`. Read that file first and confirm `get`
returns `(int, string)`; if its signature differs, adapt the calls rather than
duplicating the helper.

- [ ] **Step 2: Run it to verify it fails**

Run: `go test -race -tags purego ./internal/app/server/ -run TestMonitoring -v`
Expected: FAIL — `TestMonitoringKeysOnRouteTemplate` reports "no server_request
series", because no middleware emits one yet.

- [ ] **Step 3: Write the middleware**

Create `internal/app/server/monitoring.go`:

```go
package server

import (
	"strconv"
	"time"

	"github.com/labstack/echo/v4"
	"github.com/spacemonkeygo/monkit/v3"
)

var mon = monkit.Package()

// monitoring records one duration and one count per request, keyed by the
// route template rather than the request path.
//
// The template matters. Keying on c.Request().URL.Path would create a series
// per media id, which is unbounded, and a scanner walking the service would
// take the metrics store down with it. c.Path() is the pattern echo matched -
// "/api/v1/medias/:id" - so the cardinality is the size of the route table.
//
// This replaced an observe(operation string) helper that was copy-pasted into
// 12 slice packages and called by hand at 62 sites, which covered only the
// handlers whose author remembered. A middleware covers all of them and cannot
// be forgotten on a new route.
func monitoring() echo.MiddlewareFunc {
	return func(next echo.HandlerFunc) echo.HandlerFunc {
		return func(c echo.Context) error {
			began := time.Now()

			err := next(c)

			// c.Path() is empty when echo matched no route. Recording the real
			// URL there would be the cardinality bug this function exists to
			// avoid, so unmatched requests share one bucket.
			path := c.Path()
			if path == "" {
				path = "<unmatched>"
			}

			// Read the status after next has run: the error handler may still
			// change it, and what a dashboard needs is what the client saw.
			status := c.Response().Status

			tags := []monkit.SeriesTag{
				monkit.NewSeriesTag("path", path),
				monkit.NewSeriesTag("method", c.Request().Method),
				monkit.NewSeriesTag("status", strconv.Itoa(status)),
			}

			mon.DurationVal("server_request", tags...).Observe(time.Since(began))
			mon.Counter("server_request_count", tags...).Inc(1)

			return err
		}
	}
}
```

- [ ] **Step 4: Insert it into the chain**

In `internal/app/server/server.go`, directly after `e.Use(middleware.Recover())`
(line 129), add:

```go
	// Monitoring sits inside Recover, so a panicking handler is still counted
	// as the 500 the client received rather than vanishing from the metrics.
	e.Use(monitoring())
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `go test -race -tags purego ./internal/app/server/ -v`
Expected: PASS, including the pre-existing tests in `server_test.go`.

- [ ] **Step 6: Commit**

```bash
git add internal/app/server/monitoring.go internal/app/server/monitoring_test.go internal/app/server/server.go
git commit -m "feat(server): instrument every route with one middleware

Keyed on the route template, not the request path, so cardinality is the size
of the route table rather than the number of distinct ids seen."
```

---

### Task 4: Delete the 62 hand-written API observations

**Files:**
- Modify: `internal/app/usage/endpoint.go:227`, `internal/app/highlight/endpoint.go:368`,
  `internal/app/summary/endpoint.go:284`, `internal/app/user/endpoint.go:586`,
  `internal/app/statistic/endpoint.go:583`, `internal/app/editor/endpoint.go:316`,
  `internal/app/playlist/endpoint.go:601`, `internal/app/watermark/endpoint.go:245`,
  `internal/app/apikey/endpoint.go:251`, `internal/app/webhook/endpoint.go:339`,
  `internal/app/player/endpoint.go:611`, `internal/app/report/endpoint.go:254`
  (the 12 identical `observe` helpers)
- Modify: `internal/app/media/chapter_endpoint.go`, `internal/app/media/caption_endpoint.go`,
  `internal/app/media/media_endpoint.go`, `internal/app/live/*` (the inline
  `ApiSum` sites that have no helper)

**Interfaces:**
- Consumes: the `server_request` series from Task 3. Nothing is produced; this
  task only removes code that Task 3 made redundant.

- [ ] **Step 1: Confirm Task 3 covers what you are about to delete**

Run: `grep -rn "ApiSum" --include='*.go' . | wc -l`
Expected: 62. Record the number. Every one of these is a route that the
middleware from Task 3 now covers automatically.

- [ ] **Step 2: Delete the 12 helpers and their call sites**

For each of the 12 files, delete the helper:

```go
// observe records the handler's latency under its operation id.
func observe(operation string) func() {
	start := time.Now().UTC()

	return func() {
		metrics.DbMetricsIns.ApiSum.WithLabelValues(operation).Observe(time.Since(start).Seconds())
	}
}
```

and every `defer observe("...")()` line in that file. Find them with:

```bash
grep -rn "defer observe(" --include='*.go' internal/app/
```

- [ ] **Step 3: Delete the inline sites with no helper**

In `internal/app/media/chapter_endpoint.go`, `caption_endpoint.go`,
`media_endpoint.go` and the `live` slice, the observations are written out
directly rather than through a helper. Each looks like:

```go
	start := time.Now().UTC()
	defer func() {
		metrics.DbMetricsIns.ApiSum.WithLabelValues("CreateMediaChapter").
			Observe(time.Since(start).Seconds())
	}()
```

Delete the whole block including the `start :=` line. Locate them with:

```bash
grep -rn "ApiSum" --include='*.go' internal/app/
```

- [ ] **Step 4: Remove the imports the deletions orphaned**

Run: `goimports -local 10.0.0.50/tuan.quang.tran/vms-v2 -w internal/app/`

Then confirm no slice still imports the metrics package:

```bash
grep -rn "utils/metrics" --include='*.go' internal/app/
```

Expected: no output.

- [ ] **Step 5: Verify**

Run: `go build -tags purego ./... && go test -race -tags purego ./internal/app/...`
Expected: builds clean, tests pass.

Run: `grep -rn "ApiSum" --include='*.go' . | wc -l`
Expected: 0.

- [ ] **Step 6: Commit**

```bash
git add internal/app
git commit -m "refactor(app): drop the hand-written API latency observations

62 call sites and 12 identical copies of an observe helper, replaced by the
one middleware that covers every route including the ones nobody remembered
to instrument."
```

---

### Task 5: Instrument the repositories, first pass

This task converts five representative files and, more importantly, builds the
test that catches the one non-mechanical trap in the whole migration. Task 6
does the remaining thirty by repeating the pattern proven here.

**Files:**
- Create: `pkg/v1/repositories/monkit.go`
- Create: `pkg/v1/repositories/monkit_test.go`
- Modify: `pkg/v1/repositories/api_key.repository.go`,
  `live_stream_key.repository.go`, `ai_task.repository.go`,
  `video.repository.go`, `usage.repository.go`

**Interfaces:**
- Produces: `var mon = monkit.Package()` at package scope in
  `pkg/v1/repositories`, shared by every file in the package. Task 6 uses it
  and must not declare a second one.

- [ ] **Step 1: Write the failing test**

Create `pkg/v1/repositories/monkit_test.go`. Note this is package
`repositories`, not `repositories_test`: it swaps the package-level `mon`, so
it must live inside the package.

```go
package repositories

import (
	"context"
	"errors"
	"testing"

	"github.com/spacemonkeygo/monkit/v3"
)

// withTestMon points the package scope at a registry of this test's own, so
// assertions see only what this test produced and parallel packages cannot
// bleed into the counts. The original is restored on cleanup.
func withTestMon(t *testing.T) *monkit.Registry {
	t.Helper()

	original := mon
	registry := monkit.NewRegistry()
	mon = registry.ScopeNamed("repositories")

	t.Cleanup(func() { mon = original })

	return registry
}

// counts reads the success and error counts monkit recorded for a task.
func counts(registry *monkit.Registry, task string) (success, failure float64) {
	registry.Stats(func(key monkit.SeriesKey, field string, val float64) {
		if key.Measurement != task {
			return
		}

		switch field {
		case "success":
			success = val
		case "error":
			failure = val
		}
	})

	return success, failure
}

// instrumented is the exact shape every converted repository method has. It
// exists so the trap below is asserted against the pattern rather than against
// one particular repository, whose failure would need a database.
func instrumented(ctx context.Context, fail bool) (err error) {
	defer mon.Task()(&ctx)(&err)

	if fail {
		return errors.New("boom")
	}

	return nil
}

// TestTaskRecordsFailure is the regression test for the one part of this
// migration that is not mechanical.
//
// A method with a named err return whose body writes
// `if err := f(); err != nil { return err }` declares a NEW err inside the if,
// leaves the named one nil, and so reports a success on every failed call.
// The metric is then not merely missing, it is wrong, and nothing about the
// code looks incorrect. Every conversion must assign the outer err.
func TestTaskRecordsFailure(t *testing.T) {
	registry := withTestMon(t)

	if err := instrumented(context.Background(), true); err == nil {
		t.Fatal("instrumented() = nil, want an error")
	}

	success, failure := counts(registry, "instrumented")

	if failure != 1 {
		t.Errorf("error count = %v, want 1", failure)
	}

	if success != 0 {
		t.Errorf("success count = %v, want 0; a shadowed err reported a failed call as a success", success)
	}
}

// TestTaskRecordsSuccess is the other half: a call that works is counted once
// as a success and never as an error.
func TestTaskRecordsSuccess(t *testing.T) {
	registry := withTestMon(t)

	if err := instrumented(context.Background(), false); err != nil {
		t.Fatalf("instrumented() = %v, want nil", err)
	}

	success, failure := counts(registry, "instrumented")

	if success != 1 {
		t.Errorf("success count = %v, want 1", success)
	}

	if failure != 0 {
		t.Errorf("error count = %v, want 0", failure)
	}
}
```

- [ ] **Step 2: Run it to verify it fails**

Run: `go test -race -tags purego ./pkg/v1/repositories/ -run TestTask -v`
Expected: FAIL — `undefined: mon`, because `monkit.go` does not exist yet.

If it instead fails on the field names `"success"` / `"error"`, print what
`registry.Stats` actually yields and correct the two case labels to the real
field names before continuing. Do not loosen the assertion.

- [ ] **Step 3: Create the package scope**

Create `pkg/v1/repositories/monkit.go`:

```go
package repositories

import "github.com/spacemonkeygo/monkit/v3"

// mon is the package's monkit scope, shared by all 35 repository files.
//
// It is a var rather than a const-like call inside each file so a test can
// point it at a registry of its own; see monkit_test.go.
var mon = monkit.Package()
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `go test -race -tags purego ./pkg/v1/repositories/ -run TestTask -v`
Expected: PASS, both tests.

- [ ] **Step 5: Convert `api_key.repository.go`**

Every method in the file follows one shape. Before:

```go
func (r *apiKeyRepository) CreateApiKey(
	ctx context.Context, newApiKey *domain.ApiKey,
) (*domain.ApiKey, error) {
	t := time.Now().UTC()
	defer func() {
		metrics.DbMetricsIns.DbSum.WithLabelValues("CreateApiKey").
			Observe(time.Since(t).Seconds())
	}()

	if err := r.db.WithContext(ctx).Create(newApiKey).Error; err != nil {
		return nil, err
	}

	return newApiKey, nil
}
```

After:

```go
func (r *apiKeyRepository) CreateApiKey(
	ctx context.Context, newApiKey *domain.ApiKey,
) (_ *domain.ApiKey, err error) {
	defer mon.Task()(&ctx)(&err)

	if err = r.db.WithContext(ctx).Create(newApiKey).Error; err != nil {
		return nil, err
	}

	return newApiKey, nil
}
```

Four edits per method, in this order:

1. Name the results: `(*domain.ApiKey, error)` becomes `(_ *domain.ApiKey, err error)`.
   A method returning only `error` becomes `(err error)`, never `(_ error)`.
   A method returning three values becomes `(_ A, _ B, err error)`.
2. Delete the `t := time.Now().UTC()` line and the whole `defer func(){...}()`
   block.
3. Insert `defer mon.Task()(&ctx)(&err)` as the first statement.
4. **Un-shadow every inner `err`.** Change `if err := ...; err != nil` to
   `err = ...` followed by `if err != nil`, so the named return is the one that
   gets assigned. This is the step the test in Step 1 exists to catch, and the
   only step where a careless pass produces metrics that are silently wrong.

The `"CreateApiKey"` label disappears with no loss: monkit reads the function
name off the stack, and every existing label already equals the method name.

Do this file with Bash/python edits, not the Edit tool - the formatter hook
reflows whole files.

- [ ] **Step 6: Repeat for four more files**

Apply the identical treatment to:
- `pkg/v1/repositories/live_stream_key.repository.go`
- `pkg/v1/repositories/ai_task.repository.go`
- `pkg/v1/repositories/video.repository.go`
- `pkg/v1/repositories/usage.repository.go`

`video.repository.go` is the largest and has methods returning three values;
handle those with `(_ A, _ B, err error)`.

- [ ] **Step 7: Verify the five files**

Run: `go build -tags purego ./pkg/v1/repositories/`
Expected: clean.

Run: `go vet -tags purego ./pkg/v1/repositories/`
Expected: clean. `vet` catches a `defer` on an unnamed return.

Then check no shadow survived in the converted files:

```bash
grep -n "if err :=" pkg/v1/repositories/api_key.repository.go \
  pkg/v1/repositories/live_stream_key.repository.go \
  pkg/v1/repositories/ai_task.repository.go \
  pkg/v1/repositories/video.repository.go \
  pkg/v1/repositories/usage.repository.go
```

Every hit is a place where the named `err` stays nil on failure. There must be
none inside a method that carries a `mon.Task()`.

- [ ] **Step 8: Commit**

```bash
git add pkg/v1/repositories/monkit.go pkg/v1/repositories/monkit_test.go \
  pkg/v1/repositories/api_key.repository.go \
  pkg/v1/repositories/live_stream_key.repository.go \
  pkg/v1/repositories/ai_task.repository.go \
  pkg/v1/repositories/video.repository.go \
  pkg/v1/repositories/usage.repository.go
git commit -m "refactor(repositories): instrument with monkit tasks, first pass

The histograms this replaces recorded latency and nothing else, so a failed
query was indistinguishable from a successful one. A task records both, plus
panics. Includes the regression test for the shadowed-err trap the conversion
can silently walk into."
```

---

### Task 6: Instrument the remaining thirty repository files

**Files:**
- Modify: the other 30 files in `pkg/v1/repositories/` that still reference
  `metrics.DbMetricsIns`

**Interfaces:**
- Consumes: `mon` from Task 5. Do **not** declare another package scope.

- [ ] **Step 1: List what is left**

Run:

```bash
grep -rl "DbMetricsIns" --include='*.go' pkg/v1/repositories/
```

Record the list. Expect 30 files.

- [ ] **Step 2: Convert them, five at a time**

Apply the exact four edits from Task 5 Step 5 to each file. After every group
of five:

```bash
go build -tags purego ./pkg/v1/repositories/ && go vet -tags purego ./pkg/v1/repositories/
```

Working in small groups keeps a build break attributable to five files rather
than thirty.

- [ ] **Step 3: Verify no site remains**

Run: `grep -rn "DbMetricsIns" --include='*.go' .`
Expected: only `internal/utils/metrics/metrics.go`, which Task 7 deletes.

Run: `grep -rn "if err :=" pkg/v1/repositories/`
Every hit must be inside a method with no `mon.Task()`, or it is the shadow
bug. Check each one.

- [ ] **Step 4: Run the full suite**

Run: `make test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add pkg/v1/repositories
git commit -m "refactor(repositories): instrument the remaining thirty files

Completes the 345-site conversion from prometheus histograms to monkit tasks."
```

---

### Task 7: Delete the metrics package, the dead Loki client and the deps

**Files:**
- Delete: `internal/utils/metrics/` (whole directory)
- Delete: `internal/utils/log/client.go`
- Modify: `go.mod`, `go.sum`
- Modify: `.golangci.yml:164`

**Interfaces:** nothing produced; this removes what Tasks 3 through 6 made
unreachable.

- [ ] **Step 1: Confirm nothing imports the metrics package**

Run: `grep -rn "utils/metrics" --include='*.go' .`
Expected: no output. If there is any, go back and finish Task 4 or Task 6.

- [ ] **Step 2: Confirm the Loki client is dead**

Run: `grep -rn "NewLokiClient\|LokiClient" --include='*.go' .`
Expected: only `internal/utils/log/client.go` itself, which proves nothing
calls it. If anything else appears, stop and report it rather than deleting.

- [ ] **Step 3: Delete**

```bash
git rm -r internal/utils/metrics
git rm internal/utils/log/client.go
```

- [ ] **Step 4: Drop the dependencies**

```bash
go mod tidy
```

Then confirm both are gone:

```bash
grep -n "prometheus\|ic2hrmk/promtail" go.mod
```

Expected: no output. `github.com/prometheus/common` may remain as an indirect
dependency of something unrelated; only a direct `client_golang` requirement
must be gone. If `client_golang` survives, find its remaining importer with
`go mod why github.com/prometheus/client_golang` and remove that use.

- [ ] **Step 5: Tidy the lint comment**

In `.golangci.yml` line 164, the comment reads:

```yaml
    # -- logging: log/slog, plus graylog and promtail sinks --------------------
```

Change it to:

```yaml
    # -- logging: log/slog, plus the graylog sink ------------------------------
```

Leave every rule beneath it alone. Graylog stays: it is the live log sink,
wired in both `cmd/http/init.go` and `cmd/grpc/init.go`, and it is not part of
the Grafana/Loki stack.

- [ ] **Step 6: Verify**

Run: `make test && make lint`
Expected: tests pass; lint reports no findings beyond the branch's documented
pre-existing ones.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "chore: remove the prometheus metrics package and the dead Loki client

The metrics package has no importers left. NewLokiClient was never called by
anything in the module; removing it drops the ic2hrmk/promtail dependency."
```

---

### Task 8: Remove the monitoring stack from the repo

**Files:**
- Modify: `docker-compose.yml` (services at 167-180, 181-198, 199-209, 269-281;
  the `monitor` network at line 4)
- Delete: `deploy/observability/` (whole directory)
- Modify: `env-example/app.env`

**Interfaces:** nothing produced. Monitoring runs on a separate machine; how
`/metrics` gets collected is decided later and is out of scope.

- [ ] **Step 1: Delete the four services**

From `docker-compose.yml`, remove the `prometheus`, `grafana`, `promtail` and
`node-exporter` service blocks in full. All four carry `profiles: [monitor]`,
which makes them easy to find and confirms none of them is in the default
bring-up.

- [ ] **Step 2: Remove the now-unused network**

The `monitor` network at line 4 exists only for those services. After Step 1,
confirm nothing references it:

```bash
grep -n "monitor" docker-compose.yml
```

If the only hits are the network definition itself, delete it. If any service
still lists it, that service was missed in Step 1.

- [ ] **Step 3: Remove the grafana volume**

`grafana` mounted `./libgrafana:/var/lib/grafana`. Confirm `libgrafana` is not
referenced anywhere else and remove it from `.gitignore` if it is listed there:

```bash
grep -rn "libgrafana" . --exclude-dir=.git
```

- [ ] **Step 4: Delete the config directory**

```bash
git rm -r deploy/observability
```

- [ ] **Step 5: Update the env example**

In `env-example/app.env`, line 4 reads:

```
# rabbitmq and grafana images cannot read app.yaml. These are the variables
```

Change `rabbitmq and grafana images` to `the rabbitmq image`. Then add the new
setting near the other service-level entries:

```
# Debug listener: monkit metrics, pprof, the build stamp, health and the
# dynamic log level. Loopback by default. A deployment scraped from another
# host sets 0.0.0.0:6060 here and owns the firewall rule that keeps it private.
DEBUG_ADDR=127.0.0.1:6060
```

- [ ] **Step 6: Verify the compose file still parses**

Run: `docker compose config -q`
Expected: no output, exit 0. A dangling network reference or a bad indent from
the deletions shows up here.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "chore(deploy): drop the grafana, prometheus, loki and node-exporter stack

Monitoring moves to a separate host. The service exposes /metrics on the debug
port; what scrapes it is decided outside this repo."
```

---

### Task 9: End-to-end verification

**Files:** none modified. This task proves the migration and produces the
evidence for the branch.

- [ ] **Step 1: Static checks**

```bash
cd /home/tuan/.treehouse/aioz-stream-370b08/2/aioz-stream
go build -tags purego ./...
go vet -tags purego ./...
make test
make lint
```

All four clean. `make lint` may show the branch's documented pre-existing
findings; it must show no new ones.

- [ ] **Step 2: Prove the old surface is gone**

```bash
grep -rn "utils/metrics" --include='*.go' .          # want: no output
grep -rn "DbMetricsIns\|ApiSum\|DbSum" --include='*.go' .   # want: no output
grep -n "client_golang\|ic2hrmk" go.mod              # want: no output
ls deploy/observability 2>&1                          # want: No such file or directory
```

- [ ] **Step 3: Boot and probe the debug port**

Start `cmd/http` against your local `debug.env`, then:

```bash
curl -sf http://127.0.0.1:6060/metrics | head -20
curl -sf http://127.0.0.1:6060/version/
curl -sf http://127.0.0.1:6060/health
curl -sf 'http://127.0.0.1:6060/debug/pprof/goroutine?debug=1' | head -5
curl -sf http://127.0.0.1:6060/top
```

`/metrics` must be non-empty **before you make any API request**. That is the
proof `monitor.Register` ran; an empty body here means an idle service will
look dead to whatever scrapes it.

- [ ] **Step 4: Prove the public port no longer serves them**

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:<API_PORT>/metrics   # want 404
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:<API_PORT>/version   # want 404
```

- [ ] **Step 5: Prove the middleware works against the real service**

Make one real API call, then:

```bash
curl -sf http://127.0.0.1:6060/metrics | grep server_request | head -10
```

The tag must be the route template (`path="/api/v1/medias/:id"`), never a
concrete id. A concrete id here is the cardinality bug and blocks the branch.

- [ ] **Step 6: Prove error counting works end to end**

Make an API call that fails in a repository - a lookup for an id that does not
exist is enough - then:

```bash
curl -sf http://127.0.0.1:6060/metrics | grep -i 'error' | head -20
```

A non-zero error field on the repository series proves the named `err` is
actually assigned. If every series shows successes and no errors while the API
returned a failure, the shadowed-`err` trap survived somewhere in Task 5 or 6:
find it before merging, because the metric is wrong rather than missing.

- [ ] **Step 7: Commit anything outstanding and push**

```bash
git status                      # expect clean
git push origin refactor/sync-with-template-source
```

## Self-Review Notes

Checked against the spec:

- Section 1 (`internal/app/debug`) → Tasks 1 and 2.
- Section 2 (345 `DbSum` sites) → Tasks 5 and 6.
- Section 3 (62 `ApiSum` sites) → Tasks 3 and 4.
- Section 4 (removals) → Tasks 7 and 8.
- Spec's verification list → Task 9, every item.
- Spec's "shadowed `err`" trap → Task 5 Step 1 is a dedicated regression test,
  re-checked in Task 6 Step 3 and Task 9 Step 6.
- Spec's "kept: Graylog" → Task 7 Step 5 states it explicitly.
- Spec's follow-ups (`cmd/grpc`, `aioz-stream-core`, collection, aioz-common
  changes) → deliberately absent; they are out of scope.

Names used consistently across tasks: `debug.New`, `debug.DefaultConfig`,
`debug.Config.Addr`, `(*Server).Name/Run/Close/Addr`, `monitoring()`,
`mon`, `withTestMon`, series `server_request` and `server_request_count`.
