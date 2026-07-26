---
type: plan
project: depin
created: 2026-07-09
tags: [coord, metrics, monkit, telemetry, prometheus, remote-write]
---

# Coordinator metrics collection (push via Prometheus remote_write)

## Context

The coordinator already has a **complete metrics pipeline** — it was ported from Storj and never stripped:

- `monkit/v3` is a direct dep; `mon.Task()` tracing is pervasive (81 files).
- `pkg/debug/server.go` exposes `/metrics` (Prometheus exposition of `monkit.Default`) plus `/mon/`, `/top`, pprof, `/health`.
- Every coord role wires it in via `base.setupDebug()` (`coord/peer.go:225-249`), gated only on `Config.Debug.Addr != ""`.

Two things block "metrics from every component" end-to-end:

1. **No exporter to the VPS.** `/metrics` exists but only on a random loopback port (`Debug.Addr = "127.0.0.1:0"`, `pkg/debug/server.go:28`), and there is no path off-box. There is **no telemetry client** in the repo — the only `telemetry` mention is a comment (`coord/peer.go:508`); `storj.io/private/telemetry` / `process` are not deps.
2. **Uneven coverage.** Tracing is everywhere, but bespoke business counters exist only in a few packages (`coord/rangedloop/stats.go:149`, `coord/relay/observer.go:181`, `coord/reputation`, `coord/contact`, `internal/p2pmonitor`). Audit outcomes, repair queue depth/success, order signing, file create/commit, tally, GC, expired-deletion emit no domain counters yet.

Goal (confirmed with user): **Both** — get metrics off every coord role AND fill instrumentation gaps. **Collection model = push via Prometheus `remote_write`** (user's choice): each role walks `monkit.Default` on an interval, converts the stats to Prometheus TimeSeries, and **POSTs them outbound** to a Prometheus remote-write receiver on a **separate Prometheus/Grafana VPS**. Push means **no inbound port** on coord → no debug-server exposure, no firewall carve-outs; scaled/ephemeral replicas each push under their own `instance` label with zero service discovery; and it lands **directly in Prometheus** with no statsd_exporter/pushgateway middle layer. The debug `/metrics` endpoint stays as-is on loopback for local ops.

> Note: `docker-compose.coord-split.yml` (per-role containers) is **not on this HEAD** — it lives on the split branch (see memory `coord-split-compose`). Execution targets that branch/file for env wiring. Roles: `api`, `core`, `ranged-loop`, `audit`, `repair`, `relay` (+ `run` all-in-one for dev).

---

## Part A — remote_write push client (the port)

Reuse the exact monkit stat-walk already in `pkg/debug/prometheus.go:91`, but instead of writing text to an HTTP response, build Prometheus `TimeSeries` and POST them to a remote-write receiver.

**A1. New package `pkg/telemetry`:**
- `Config` (flag-tagged): `URL string` (`default:""`, empty = disabled — off-by-default like Storj `metrics.addr`; the receiver, e.g. `https://vps:9090/api/v1/write`), `App string` (`default:"coord"`), `Instance string` (`default:""` → resolved to hostname/container id at start so replicas separate), `Role string` (`default:""` → set per role via env), `Interval time.Duration` (`default:"15s"`), and optional receiver auth: `BearerToken`/`Username`+`Password` (`user:"true"`).
- `Client{ log, http *http.Client, registry *monkit.Registry, cfg, extraLabels }`; `NewClient(log, cfg, registry)`.
- `Run(ctx)`: ticker on `Interval`; each tick `report(ctx)`. `Close()` no-op/cancel. Signature matches `lifecycle.Item{Run, Close}`.
- `report(ctx)`: walk `registry.Stats(func(key monkit.SeriesKey, field string, val float64))` (copy `prometheus.go:91-110`); for each build a `TimeSeries{Labels: [__name__=sanitize(measurement), field=field, role, instance, app, ...key.Tags], Samples: [{Value: val, Timestamp: time.Now().UnixMilli()}]}`. Marshal a `WriteRequest`, **snappy-compress** (`github.com/golang/snappy`), HTTP `POST cfg.URL` with headers `Content-Encoding: snappy`, `Content-Type: application/x-protobuf`, `X-Prometheus-Remote-Write-Version: 0.1.0` (+ auth header if set). Chunk into multiple requests if the series count is large.
- **Protobuf dep:** prefer a small remote-write client rather than pulling the whole `github.com/prometheus/prometheus` module. Options (decide at execution): (a) tiny lib `github.com/castai/promwrite` (WriteRequest/TimeSeries/Label/Sample + snappy + POST in ~one file); (b) generate the 4 `prompb` messages (`WriteRequest`,`TimeSeries`,`Label`,`Sample`) via the repo's existing `buf` toolchain. `prometheus/common` + `client_model` are already indirect deps. Recommend (a) for maintainability.
- Register process/runtime metrics once at startup via `environment.Register(monkit.Default)` (`github.com/spacemonkeygo/monkit/v3/environment`) so **CPU (getrusage user/sys), memory (Go MemStats + max RSS), goroutines, GC, fds** ride along like Storj.

**A2. Wire into every role, once.** All six role peers already call `base.setupDebug()` (`coord/{peer,api,core,audit_peer,rangeloop,repair_peer}.go`). Add `b.setupTelemetry()` and call it from the end of `setupDebug()` (`coord/peer.go:250`) — single edit covers every role. It builds the client from `b.Config.Metrics` against `monkit.Default` and, only if `URL != ""`, registers `lifecycle.Item{Name:"telemetry", Run: client.Run, Close: client.Close}` (same pattern as the debug item, peer.go:245). Relay (`coord/relay/peer.go`) has its own `setupDebug` — add the same call there.
- Add `Metrics telemetry.Config` to the coord `Config` struct (`coord/config.go:34`, next to `Debug`). Viper env then `AIOZ_METRICS_URL`, `AIOZ_METRICS_ROLE`, etc. (top-level, not under `app-config`; verify with `bin/coord api --help | grep metrics` per the DSN gotcha in memory).

**A3. Enable per role via env in the split compose** — set `AIOZ_METRICS_URL=https://<vps>:9090/api/v1/write` and `AIOZ_METRICS_ROLE=<role>` on each role service (`api`, `core`, `ranged-loop`, `audit`, `repair`, `relay`). No `ports:` — traffic is outbound only.

**A4. Push compatibility (critical — do not copy the pull path blindly).** The existing `/metrics` endpoint applies a **`DeltaTransformer`** (`pkg/debug/prometheus.go:16-18`, one per `output-id`) and marks everything `# TYPE gauge` (`:113`). That is right for pull, **wrong for remote_write**. Requirements for the push client:
- **Read the base registry directly — NO DeltaTransformer.** Send monkit's **cumulative** values (a `Counter`/`Meter`'s running `count`/`total`, not the since-last-read delta), so Prometheus `rate()`/`increase()` and counter-reset detection work across the full retention. Applying the delta transformer would make every counter meaningless under push.
- **Stamp `time.Now().UnixMilli()`** on every sample and attach **stable `role` + `instance` labels** to every series, so replicas keep a constant identity across pushes and don't collide or churn (remote_write rejects out-of-order/duplicate series).
- **Counter vs gauge:** monkit exports all fields as raw floats with a `field` label (`count`,`sum`,`min`,`max`,`avg`,`recent`,`rXX` quantiles for distributions/`mon.Task`; `value` for `IntVal`; `count`/`total` for `Counter`/`Meter`). Push them as-is (Prometheus treats unknown-typed samples fine); optionally suffix monotonic counter series with `_total` and rely on `rate()`. Process restart resets to 0 — acceptable; note it in the dashboard queries.
- **Cardinality control:** `mon.Task` emits `function{...}`/`function_times{...}` for **every** instrumented function × many quantile fields — at a 15s interval that is a large series volume over remote_write. Add a `Config` include/exclude filter (measurement prefix allow/deny, and a "drop quantiles" toggle) so an operator can push just counters + env + the Part-C business metrics and skip the per-function firehose if needed. Default: include business + env, sample-limit the `function_times` quantiles.
- **Reuse `sanitize()`** from `pkg/debug/prometheus.go` for metric-name/label validity; `__name__` must match `[a-zA-Z_:][a-zA-Z0-9_:]*`.
- **Backpressure:** POST has a timeout; on receiver error, log + drop that interval (never block the lifecycle goroutine or buffer unboundedly).

---

## Part B — VPS setup + dashboards (delivery)

Ship as artifacts under `monitoring/` (applied on the VPS, nothing runs from this repo):

- `monitoring/README.md` — VPS steps: run Prometheus with **`--web.enable-remote-write-receiver`** (exposes `/api/v1/write`); secure it (reverse-proxy with TLS + basic-auth / bearer, matching the coord `Metrics` auth fields); optionally set `out_of_order_time_window` if replicas' clocks drift. Lists the exact `AIOZ_METRICS_*` env each coord role needs.
- `monitoring/grafana/dashboards/*.json` — importable dashboards (datasource templated as a variable):
  - `coord-overview.json` — role up/heartbeat, gRPC rate+latency (`function`/`function_times` series), goroutines, DB pool, CPU (rusage) + process mem/GC.
  - `coord-storage-lifecycle.json` — file create/commit/download, order signing, tally, expired-deletion, GC.
  - `coord-integrity.json` — audit pass/fail/offline/contained, reputation disqual/suspend, repair queue depth + repair success/fail, ranged-loop pass totals.

No Prometheus/Grafana services are added to any compose file in this repo.

---

## Part C — Fill instrumentation gaps (per component)

Pattern for every gap: reuse the existing package-level `var mon = monkit.Package()` (add one where missing, e.g. `coord/<pkg>/monkit.go` — see `coord/placement/monkit.go`), then emit at the business event using the same primitives already in the codebase:
- counters/rates → `mon.Counter("name").Inc(1)` or `mon.Meter(...)`,
- gauges/levels → `mon.IntVal(...).Observe(v)` (see `coord/rangedloop/stats.go:149`),
- durations → `mon.DurationVal(...).Observe(d)` (see `coord/rangedloop/service.go:308`),
- discrete events → `mon.Event(...)` (see `coord/reputation/service.go:76`).
Emit at the **outcome** site (after success/failure known), not just as `mon.Task` entry.

Priority targets (component | file | metrics to add):

| Component         | File                                        | Add                                                                                           |
| ----------------- | ------------------------------------------- | --------------------------------------------------------------------------------------------- |
| audit worker      | `coord/audit` (`NewWorker`)                 | `audit_segments_{success,failed,offline,contained,unknown}` counters, verify duration         |
| audit reverify    | `coord/audit` (`NewReverifyWorker`)         | reverify processed/pass/fail counters                                                         |
| repair checker    | `coord/repair/checker/observer.go`          | `repair_segments_{healthy,injured,clumped,over_threshold}`, enqueued count                    |
| repair queue      | `coord/repair/queue/queue.go`               | `repair_queue_length` gauge (on select/insert)                                                |
| repair worker     | `coord/repair/repairer/repairer.go`         | `repair_{success,failed}`, pieces reconstructed, `repair_strategy{rs,clone}`, repair duration |
| order signing     | `coord/order/endpoint.go`                   | limits signed by action (upload/download), signing errors                                     |
| orders settlement | `coord/order` (`OrdersService`)             | settled bytes + rows recorded                                                                 |
| file metadata     | `coord/file/endpoint.go`                    | segment create/commit/download counts, inline vs remote, bytes                                |
| accounting tally  | `coord/accounting/tally/service.go`         | tally cycle duration, objects/bytes tallied                                                   |
| worker tally      | `coord/accounting/workertally/service.go`   | workers tallied, payout bytes                                                                 |
| overlay caches    | `coord/overlay` (upload/download selection) | reputable/new/eligible node gauges, refresh duration                                          |
| expired-deletion  | `coord/file/expireddeletion/chore.go`       | rows deleted per cycle                                                                        |
| GC bloomfilter    | `coord/gc/bloomfilter/observer.go`          | bundles built, pieces per filter, filter size                                                 |
| GC sender         | `coord/gc/sender/service.go`                | Retain RPCs sent success/fail                                                                 |

Skip endpoints whose only useful metric is request count/latency — `mon.Task()` already yields that at `/metrics` (`function`/`function_times` series). Also leave the `eventkit` stub (`coord/accounting/tally/eventkit.go`) as-is; out of scope.

---

## Critical files

- `pkg/telemetry/**` — **new**: `client.go` (remote_write push loop: `registry.Stats` walk from `pkg/debug/prometheus.go:91` → `TimeSeries` → snappy → POST), `config.go`.
- `coord/peer.go:250` — add `setupTelemetry()` + call from `setupDebug()`; `coord/config.go:34` — add `Metrics telemetry.Config`; `coord/relay/peer.go` — same call in its debug setup.
- `pkg/debug/prometheus.go` (unchanged; source of the stat-walk to copy), `pkg/debug/server.go` (unchanged, stays on loopback).
- `docker-compose.coord-split.yml` (on split branch) — add `AIOZ_METRICS_*` env per role (no ports).
- `monitoring/**` — new (README, Grafana dashboards).
- Part C instrumentation files in the table above (reuse `var mon = monkit.Package()`; template `coord/rangedloop/stats.go`).

## Verification (end-to-end)

1. **Push works:** run a throwaway Prometheus with `--web.enable-remote-write-receiver` (or a tiny mock `/api/v1/write` that logs); set `AIOZ_METRICS_URL` to it; start `coord run` → within one interval it receives snappy-protobuf TimeSeries carrying `function_times…`, env CPU/mem series, and the new counters, labeled `role`/`instance`.
2. **In Prometheus:** query the receiver → series present; `sum by(role,instance)(...)` separates roles and scaled replicas; CPU (`environment_rusage…`) and mem (`environment_runtime…`) non-zero. **Push-correctness check:** a counter series climbs **monotonically** across successive pushes (cumulative), NOT flat per-interval deltas → `rate(<counter>[1m])` is stable and sane, proving the DeltaTransformer was correctly omitted.
3. **Instrumentation firing:** run a real workload (uplink upload + download; trigger a `coord ranged-loop` pass and `coord audit`/`coord repair`) so audit/repair/file/order/tally counters increment. Re-query (`audit_segments_success`, `repair_queue_length`, `file_segment_commit`, …) → non-zero.
4. **Dashboards:** import the three JSONs into the VPS Grafana → panels render live for every subsystem.
5. **Off-by-default safety:** with `AIOZ_METRICS_URL` unset, no telemetry item is registered and behavior is unchanged; `go build ./...` and `go test ./coord/... ./pkg/telemetry/...` green. Watch the formatter-loop hazard (memory) after edits inside scan/observer loops.

## Post-approval (not part of plan mode)

Per global rules, after approval dispatch the hermes CLI to write this plan to `projects/depin/plans/2026-07-09-coord-metrics-collection.md` and refresh `projects/depin/INDEX.md`.
