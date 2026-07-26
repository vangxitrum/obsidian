# Relay observability: RTT + rejection reasons + circuit lifetime + relayed rate

Date: 2026-07-14
Status: approved (design), pending implementation plan

## Context

The coord relay (`coord/relay/`) is a standalone libp2p circuit-relay-v2 server that
NAT'd workers reserve on, and the uplink client/coord dial workers through. In the
current deployment the relay sits on a remote (WAN) host, so every piece byte between a
local client and local workers hairpins through it. Diagnosing throughput problems showed
the relay is the dominant bottleneck, yet the relay exports no latency signal and lumps
all rejections into single counters, so the "how far is the relay from each worker" and
"why are dials being refused" questions can't be answered from metrics.

The relay already exports (via monkit -> remote_write -> Grafana "Coord" folder, and a
built-in status dashboard): connected peers/conns, active/total reservations and circuits,
reservation/circuit rejection totals, host bytes in/out + rate, relayed bytes (cumulative),
and per-protocol bandwidth. See `coord/relay/{observer.go,tracer.go,status.go}`.

This change adds the missing dimensions: relay->worker RTT ("distance"), rejection reasons,
circuit lifetime, and a relayed-throughput rate, and ships a provisioned Grafana dashboard.

## Goals

- Expose relay->worker RTT so the hairpin distance is visible and per-worker outliers are findable.
- Break rejections down by reason (the signal that would have made the observed dial-backoff storm obvious).
- Expose circuit lifetime distribution and a relayed-bytes/sec rate.
- Provision a Grafana "Relay" dashboard rendering the above plus the existing relay metrics.

## Non-goals

- Client/coord->relay RTT and end-to-end circuit RTT (measured on the dialing side, not the relay). Out of scope.
- Per-worker RTT time series in Prometheus (cardinality). Per-worker detail lives on the status dashboard only.
- Changing relay forwarding behavior, limits, or topology. Metrics only.

## Metric naming convention

The stack pushes via remote_write, so a monkit metric `X` arrives in Prometheus as
`X{field="...", role, instance}`. `mon.FloatVal/IntVal` expose fields like `value`/`recent`;
`mon.DurationVal`/`Task` expose `avg/min/max/count/sum` plus quantiles `r10/r50/r90/r99`.
IMPORTANT: telemetry `DropQuantiles` defaults to **true**, which strips the `rXX` quantile
fields from the push. Therefore RTT and any distribution we want percentiles for at fixed,
config-independent series are computed explicitly (see below), not left to monkit quantiles.

## Design

### 1. Relay -> worker RTT  (new `coord/relay/rttprobe.go`)

A small `pinger` unit: given the libp2p host, each call pings every currently-connected
peer using the libp2p ping protocol (`p2p/protocol/ping`, `ping.Ping(ctx, host, id)`, one
sample per peer), with bounded concurrency and a per-ping timeout, under an overall
deadline so it never overruns the monitor tick.

- Input: `host`, `concurrency` (default 16), per-ping `timeout` (default 5s), overall deadline (<= MonitorInterval).
- Output: `rttAgg{Samples, Failures, Min, Avg, P50, P90, P99, Max time.Duration}` + `map[peer.ID]time.Duration` (per-worker latest).
- Percentiles: sort the tick's samples and index (fleet size is hundreds, cheap).
- Because the relay holds a direct connection to each worker, this RTT is the relay<->worker network latency (the WAN hop, ~35ms in the current deployment).

Prometheus series (emitted from the observer as explicit `mon.FloatVal`/`IntVal`, so they
survive `DropQuantiles=true`):
`relay_worker_rtt_min_ms`, `_avg_ms`, `_p50_ms`, `_p90_ms`, `_p99_ms`, `_max_ms`,
`relay_worker_rtt_samples`, `relay_worker_rtt_failures`.

Per-worker RTTs are added to `stats.json` (aggregate) and an optional `/rtt.json`
(per-worker, gated by `ExposePeerList` like `/peers`) + rendered in `status.html`. Not pushed to Prometheus.

Verification: the ping protocol must be enabled on both hosts. libp2p enables it by default;
confirm the relay host (`coord/relay/peer.go`) and worker host (`worker/server/server.go`)
do not disable it.

### 2. Rejection reasons  (`coord/relay/tracer.go`)

`ReservationRequestHandled(status pbv2.Status)` and `ConnectionRequestHandled(status)` already
receive the status. On a non-OK status, in addition to the existing aggregate atomic counter,
increment a tagged monkit counter:
- `relay_reservation_rejections{status=<pbv2.Status.String()>}`
- `relay_circuit_rejections{status=<...>}`   (e.g. `NO_RESERVATION`, `RESOURCE_LIMIT_EXCEEDED`)

`mon` is the package-level `monkit.Package()` in `observer.go`; reuse it from `tracer.go`
(same package) and import monkit for `NewSeriesTag`. Status cardinality is small and bounded.

### 3. Circuit lifetime  (`coord/relay/tracer.go`)

`ConnectionClosed(d time.Duration)` already has the lifetime (today only debug-logged). Add
`mon.DurationVal("relay_circuit_lifetime").Observe(d)`. This yields `avg/min/max/count` in the
push always, and `r50/r90/r99` when `DropQuantiles=false`. No custom histogram.

### 4. Relayed throughput rate  (`coord/relay/observer.go`)

The observer keeps the previous `BytesRelayed` value and timestamp; each tick it emits
`relay_bytes_relayed_rate` (bytes/sec) alongside the existing cumulative `relay_bytes_relayed`.
Added to `sample`, `observe()` gauges, and the `Stats()` StatSource.

### 5. Wiring & surfacing

- `observer.go`: hold a `*pinger`, prev-relayed state, and the last RTT agg + per-worker map;
  extend `sample`, `currentSample()`, `observe()` (new gauges), and `Stats()` (new fields).
- `status.go` + `status.html`: add RTT aggregate fields to `statsJSON`; optional `/rtt.json` for
  per-worker; render an RTT section.
- `config.go`: add `RTTProbe bool default:"true"` (off switch) and `RTTProbeTimeout time.Duration default:"5s"`;
  reuse `MonitorInterval` for cadence and `MaxReservations`-style help text style. Keep it minimal.

### 6. Grafana dashboard

- New `monitoring/grafana/dashboards/relay/relay-overview.json` (uid `relay-overview-v1`, datasource `${DS_PROMETHEUS}`),
  matching the existing coord dashboards' shape. Panels:
  - RTT: `relay_worker_rtt_p50_ms|_p90_ms|_p99_ms` timeseries; `relay_worker_rtt_samples`/`_failures`.
  - Rejections by reason: `sum by (status) (rate(relay_reservation_rejections{...}[$__rate_interval]))` and the circuit variant, stacked.
  - Circuit lifetime: `relay_circuit_lifetime{field="avg"}` (+ `r90` when available).
  - Throughput: `relay_bytes_relayed_rate` + existing `relay_rate_in/out`.
  - Capacity (reuse existing): reservations/circuits active vs max, connected peers/conns.
- Add a `relay` provider to `monitoring/grafana/provisioning/dashboards/dashboards.yml` (folder "Relay",
  path `/var/lib/grafana/dashboards/relay`), mirroring the coord/edge providers. The dashboards
  dir is already mounted by docker-compose, so only the provider entry is needed.

## Files

- `coord/relay/rttprobe.go` (new) + `rttprobe_test.go` (new)
- `coord/relay/observer.go`, `tracer.go`, `status.go`, `status.html`, `config.go` (edit) + extend `observer_test.go`, `tracer_test.go`, `status_test.go`
- `monitoring/grafana/dashboards/relay/relay-overview.json` (new)
- `monitoring/grafana/provisioning/dashboards/dashboards.yml` (edit)

## Testing

- `rttprobe_test.go`: two in-memory libp2p hosts connected; assert a probe returns Samples>=1 and RTT>0; unit-test the percentile math on synthetic duration sets (incl. empty -> zero agg, single sample).
- `tracer_test.go`: feed non-OK statuses -> assert tagged rejection counters increment per status; feed `ConnectionClosed(d)` -> assert `relay_circuit_lifetime` observed.
- `observer_test.go`: two ticks with a rising `BytesRelayed` -> assert `relay_bytes_relayed_rate` ~ delta/interval; assert RTT fields flow into `sample`/`Stats()`.
- Overrun guard: assert the probe respects its overall deadline (stub a slow ping).

## Verification (end-to-end)

Run `coord relay` against the live fleet, then:
- Scrape/observe `relay_worker_rtt_p90_ms` ~ 35ms (matches the measured WAN hop); `_samples` ~ connected worker count.
- Trigger refusals (or observe under load) -> `relay_circuit_rejections{status="NO_RESERVATION"}` increments; confirm this reproduces the dial-backoff signal seen during diagnosis.
- `relay_circuit_lifetime{field="avg"}` and `relay_bytes_relayed_rate` populate.
- Grafana shows a "Relay" folder with `relay-overview` rendering all panels; the built-in status dashboard shows the RTT section.

## Status

Implemented and e2e-verified 2026-07-14 (go vet + go test ./coord/relay green; local relay + holder peer -> rtt_samples=1, rtt_p90_ms=0.41ms localhost). Not committed (no-auto-commit).
