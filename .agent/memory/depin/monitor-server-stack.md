---
type: decision
tags: [depin, monitoring, prometheus, grafana, caddy, telemetry, remote-write, docker-compose]
created: 2026-07-13
agent: claude (main thread)
---

Built the **monitor server** (receiver side) for the coord push-metrics
pipeline as a turnkey `docker-compose` stack committed under `monitoring/` in
the depin repo. Complements [[telemetry-remote-write-push]] (the client) and
[[coord-monitoring-vps-artifacts]] (the dashboards). Plan lives at
`projects/depin/plans/2026-07-13-monitor-server-hosting.md`.

## What it is (chosen with the user: Prometheus + Caddy + turnkey compose)
Three services on a separate monitoring VPS, only Caddy published to the host:
- **Prometheus** `v2.53.0` with `--web.enable-remote-write-receiver` (accepts
  `POST /api/v1/write`) + `--storage.tsdb.retention.time`. Receiver-only, minimal
  `prometheus.yml` (just a self-scrape).
- **Grafana** `11.1.0` with provisioning: a default Prometheus datasource
  (`uid: prometheus`, `isDefault: true`) + a file dashboard provider that
  auto-loads the 3 existing `grafana/dashboards/*.json` into a **Coord** folder.
  The dashboards' `DS_PROMETHEUS` template var (empty `current`, no `__inputs`)
  resolves to the default datasource, so no manual import.
- **Caddy** `2`: auto Let's Encrypt TLS + bearer-token gate. Metrics vhost
  accepts ONLY `path /api/v1/write` + `Authorization: Bearer $METRICS_BEARER_TOKEN`
  and reverse_proxies to `prometheus:9090`; everything else -> 401. Separate
  Grafana vhost. Matches the telemetry client's `BearerToken` field.

Files: `monitoring/{docker-compose.yml, .env.example, prometheus/prometheus.yml,
caddy/Caddyfile, grafana/provisioning/{datasources,dashboards}/*.yml}` + rewritten
`README.md`. Operator flow: `cp .env.example .env`, edit domains/token/admin pw,
point DNS, `docker compose up -d`. No coord/repo code changes; client wiring is
operational only (set `AIOZ_METRICS_URL`/`ROLE`/`BEARER_TOKEN` per role).

## Key gotcha found by end-to-end verification (real bug in first draft)
`out_of_order_time_window` is a **config-file setting, not a CLI flag**. Passing
`--storage.tsdb.out-of-order-time-window` makes Prometheus refuse to start
(`unknown long flag`, crash-loop). Correct form is in `prometheus.yml`:
```yaml
storage:
  tsdb:
    out_of_order_time_window: 10m
```
The old `monitoring/README.md` section 3 wrongly told operators to pass it as a
flag too - fixed in the rewrite. (Retention `--storage.tsdb.retention.time` IS a
valid flag; only OOO is config-file-only.)

## Verified end-to-end (docker compose up, then torn down)
Drove a real snappy+protobuf `remote_write` (hand-built via the repo's own
`pkg/pb/telemetry/remote/v1` + `github.com/golang/snappy`, `gogo/protobuf`) at
the receiver: push -> **204**, series queryable with `role="api"` label intact.
Caddy gate: no token / wrong token / wrong path all -> **401**; valid token +
`/api/v1/write` -> forwarded (Prometheus 400 on junk body = auth+proxy passed).
Grafana: datasource health OK, all 3 dashboards in Coord folder, Grafana vhost
-> 302 login. Local-test tips: `.localhost` domains get Caddy internal certs;
publish `prometheus:9090` via a git-ignored `docker-compose.override.yml` to push
directly and bypass the internal CA (Go http client won't trust it over HTTPS).

Not committed: nothing external published; branch is whatever was checked out
(develop). This was implemented on top of the shipped
`feat/coord-metrics-collection` work.
