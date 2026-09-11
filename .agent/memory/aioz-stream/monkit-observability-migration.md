---
type: decision
tags: [aioz-stream, aioz-common, monkit, observability, metrics, debug-server]
created: 2026-09-09
agent: main
---

Replaced aioz-stream's prometheus metrics package with `aioz-common/stats`
(monkit) on branch `refactor/sync-with-template-source` in **worktree 2**
(`~/.treehouse/aioz-stream-370b08/2/aioz-stream`), 10 commits, `bdc20fde..a000d77d`.
Continues [[template-alignment]].

**Shape:** new `internal/app/debug` wraps `aioz-common/stats/debug.Server` and
satisfies the existing `app.Chore` interface (`Name/Run/Close`), registered
first in the lifecycle group so it closes last. `/metrics`, `/version/`,
`/health`, `/top`, `/mon/`, `/logging`, pprof all move to `127.0.0.1:6060`
(`DEBUG_ADDR`); the public API serves none of them. 345 repository histogram
sites became `defer mon.Task()(&ctx)(&err)`; 144 hand-written API observation
sites (82 `defer observe()` calls + 50 inline blocks + 12 copies of an
identical `observe` helper) became one echo middleware. Net -1264 lines.
Grafana/prometheus/promtail/node-exporter and `deploy/observability/` deleted.
Lint went from 9 pre-existing findings to **0**.

**The four things that actually cost time:**

1. **The "shadowed err" trap does not exist.** The worry was that
   `if err := f(); err != nil { return err }` leaves a named `err` result nil,
   so monkit records a failed call as a success. It does not: `return err`
   assigns the named result *before* deferred functions run. Verified with a
   deliberate reproduction. So the conversion is purely mechanical - signature
   + defer only, no body rewrites. What genuinely is invisible is a method that
   swallows the error and returns nil, which is a bug in the method.

2. **A monkit task is not its own measurement.** Every task lands under the
   measurement `function`, with the function name as the **tag** `name`, and
   fields `successes` / `errors` / `panics` (not `success` / `error`).
   Latencies are a separate `function_times` measurement tagged
   `kind=success|failure`. Filtering on `key.Measurement == "<taskname>"` finds
   nothing. Dump `registry.Stats` before writing assertions.

3. **Echo runs `HTTPErrorHandler` outside the whole middleware chain.** On the
   way out of `next(c)` the response still says 200, so a metrics middleware
   reading `c.Response().Status` records every failed request as a success. Fix
   is echo's own logger idiom: call `c.Error(err)` inside the middleware, then
   read the status, then return nil.

4. **`cmd/http` does not use `internal/app/server`.** It builds its own
   `echo.New()` in `cmd/http/init.go:920`; `internal/app/server` is imported
   only by `internal/app/peer.go`, which nothing constructs. A middleware added
   to `server.New` is dead in production. Had to export `server.Monitoring()`
   and register it on the live instance too. Check this before adding anything
   to `internal/app/server`.

**Also:** `aioz-common`'s `debug.Config` is tagged `help:`/`default:` for
aioz-config, which viper cannot bind, so `internal/config` carries its own
`DebugConfig` with `mapstructure`/`env` tags. `response.RequestFailures`
(route + reason slug) was ported to monkit rather than deleted - the route
middleware records the status a client saw, not the reason the service chose.
Graylog stays; only the dead `NewLokiClient` went, taking `ic2hrmk/promtail`
with it. See [[aioz-common-shared-package-gaps]].
