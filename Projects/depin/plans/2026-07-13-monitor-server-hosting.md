---
type: plan
project: depin
created: 2026-07-13
tags: [depin, monitoring, prometheus, grafana, telemetry, remote-write]
---

# Plan: host a turnkey push-metrics monitor server

## Context

The metrics **push** side is already shipped: `pkg/telemetry` (branch
`feat/coord-metrics-collection`, MR !32) is a Prometheus `remote_write` client
wired into every coord role + relay (`api`/`core`/`run`/`ranged-loop`/`audit`/
`repair`/`relay`). Each role walks `monkit.Default` every 15s and POSTs
snappy+protobuf `remote_write` to `AIOZ_METRICS_URL` (a `/api/v1/write`
endpoint), authenticating via `AIOZ_METRICS_BEARER_TOKEN`. It is off when the
URL is empty. Workers do **not** push (only pull `/metrics`).

The **receive** side does not exist yet. `monitoring/` holds only `README.md`
(prose describing an external Prometheus VPS) and 3 Grafana dashboard JSONs.
Nothing is runnable — an operator today would have to hand-assemble Prometheus
+ Grafana + a reverse proxy from the README's code snippets.

**Goal:** materialize a turnkey `docker-compose` monitor stack under
`monitoring/` so an operator can `cp .env.example .env`, edit it, and
`docker compose up -d` on a separate monitoring VPS to get a secured
remote_write receiver + Grafana with the dashboards auto-loaded. No coord/repo
code changes — the client side is done; only the receiver is new.

Decisions (chosen with the user): **backend = Prometheus** with
`--web.enable-remote-write-receiver` (matches existing README + dashboards);
**auth/TLS = Caddy** bearer-token check + automatic Let's Encrypt (matches the
telemetry client's `BearerToken` field); **deliverable = turnkey compose
stack** committed under `monitoring/`.

## What already exists (do NOT rebuild)

- `pkg/telemetry/{client.go,config.go}` — the push client. No changes.
- `monitoring/grafana/dashboards/{coord-overview,coord-storage-lifecycle,coord-integrity}.json`
  — reuse as-is. Each has a `DS_PROMETHEUS` datasource template var (empty
  `current`, `refresh: 1`, no `__inputs`), so it resolves to whatever
  Prometheus datasource is marked **default**. Do not edit these files.
- `monitoring/README.md` — rewrite (see below), keep the deep sections.
- Repo compose files (`docker-compose*.yml`, `dev/coord/…`) are intentionally
  left untouched — the monitor stack is a *separate* VPS. Do not wire metrics
  env into them; document the client wiring in the README instead.

## Deliverable: new files under `monitoring/`

```
monitoring/
  docker-compose.yml
  .env.example
  prometheus/prometheus.yml
  caddy/Caddyfile
  grafana/provisioning/datasources/prometheus.yml
  grafana/provisioning/dashboards/dashboards.yml
  grafana/dashboards/*.json         (existing, mounted into Grafana)
  README.md                          (rewritten to "run this")
```

### 1. `monitoring/docker-compose.yml`

Three services on an internal network; only Caddy publishes host ports 80/443.
Prometheus and Grafana get **no** host ports (reachable only via Caddy and the
compose network).

```yaml
services:
  prometheus:
    image: prom/prometheus:v2.53.0
    restart: unless-stopped
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --web.enable-remote-write-receiver
      - --storage.tsdb.retention.time=${PROM_RETENTION:-30d}
      - --storage.tsdb.out-of-order-time-window=${PROM_OOO_WINDOW:-10m}
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus

  grafana:
    image: grafana/grafana:11.1.0
    restart: unless-stopped
    depends_on: [prometheus]
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GF_SECURITY_ADMIN_PASSWORD:?set in .env}
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_SERVER_ROOT_URL: https://${GRAFANA_DOMAIN}
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro

  caddy:
    image: caddy:2
    restart: unless-stopped
    depends_on: [prometheus, grafana]
    ports: ["80:80", "443:443"]
    environment:
      METRICS_DOMAIN: ${METRICS_DOMAIN}
      GRAFANA_DOMAIN: ${GRAFANA_DOMAIN}
      METRICS_BEARER_TOKEN: ${METRICS_BEARER_TOKEN:?set in .env}
      ACME_EMAIL: ${ACME_EMAIL}
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy-data:/data
      - caddy-config:/config

volumes:
  prometheus-data:
  grafana-data:
  caddy-data:
  caddy-config:
```

### 2. `monitoring/prometheus/prometheus.yml`

Minimal — data arrives by push, so the only scrape target is Prometheus's own
self-metrics.

```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]
```

### 3. `monitoring/caddy/Caddyfile`

Auto-TLS via Let's Encrypt (`ACME_EMAIL`). Metrics vhost accepts only
`POST /api/v1/write` with the correct bearer token; everything else → 401.
Grafana vhost proxies to Grafana (which enforces its own login).

```
{
    email {$ACME_EMAIL}
}

{$METRICS_DOMAIN} {
    @write {
        path /api/v1/write
        header Authorization "Bearer {$METRICS_BEARER_TOKEN}"
    }
    handle @write {
        reverse_proxy prometheus:9090
    }
    handle {
        respond "Unauthorized" 401
    }
}

{$GRAFANA_DOMAIN} {
    reverse_proxy grafana:3000
}
```

### 4. `monitoring/grafana/provisioning/datasources/prometheus.yml`

Provision the Prometheus datasource as **default** so the dashboards'
`DS_PROMETHEUS` variable resolves without manual import.

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    uid: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

### 5. `monitoring/grafana/provisioning/dashboards/dashboards.yml`

Auto-load the existing dashboard JSONs from the mounted dir.

```yaml
apiVersion: 1
providers:
  - name: coord
    orgId: 1
    folder: Coord
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: false
```

### 6. `monitoring/.env.example`

```
# Public DNS names (A records must point at this VPS before first boot; Caddy
# provisions TLS for them via Let's Encrypt)
METRICS_DOMAIN=metrics.example.com
GRAFANA_DOMAIN=grafana.example.com
ACME_EMAIL=ops@example.com

# Shared secret: every coord role sends this as AIOZ_METRICS_BEARER_TOKEN.
# Generate with: openssl rand -hex 32
METRICS_BEARER_TOKEN=change-me-to-a-long-random-token

# Grafana admin login
GF_SECURITY_ADMIN_PASSWORD=change-me

# Optional tuning
PROM_RETENTION=30d
PROM_OOO_WINDOW=10m
```

### 7. Rewrite `monitoring/README.md`

- Rewrite sections **1–2** from "here's an example to copy" into "run the
  stack in this directory": `cp .env.example .env`, edit domains + token +
  admin password, point DNS at the VPS, `docker compose up -d`. Explain the
  three services and that Prometheus/Grafana are not exposed directly (only via
  Caddy).
- Replace section **5** (manual dashboard import) with "dashboards are
  auto-provisioned via `grafana/provisioning/`; they appear in the **Coord**
  folder on first boot." Keep the manual-import note as a fallback.
- **Keep** sections 3 (clock drift / out-of-order window), 4 (the
  `AIOZ_METRICS_*` env table), 6 (monkit field convention + PromQL examples),
  7 (process/runtime stats) — they are accurate and valuable.
- In section 4, make the client wiring concrete: each coord role/process sets
  `AIOZ_METRICS_URL=https://$METRICS_DOMAIN/api/v1/write`,
  `AIOZ_METRICS_BEARER_TOKEN=<same token as .env>`, and a **distinct**
  `AIOZ_METRICS_ROLE` (`api`/`core`/`run`/`ranged-loop`/`audit`/`repair`/
  `relay`). Show a per-role snippet the operator drops into whatever runs each
  coord role (systemd unit env, the coord-split compose service env, k8s env),
  and note the repo's own compose files are deliberately not modified.

## Client side (coord)

No code changes. The push client and its config already exist. The only action
is operational: set the three `AIOZ_METRICS_*` env vars (URL, bearer token,
role) on each running coord role/replica — documented in README section 4.

## Verification (end-to-end, real push)

Do this from the repo with the stack running locally. For local testing,
override the public domains with a local override so Caddy issues an internal
cert (or publish Prometheus directly for the curl checks):

1. **Stack health**
   - `cd monitoring && cp .env.example .env` (set a token + admin password;
     for local use set `METRICS_DOMAIN=metrics.localhost`,
     `GRAFANA_DOMAIN=grafana.localhost`).
   - `docker compose up -d && docker compose ps` — all three `Up`/healthy.

2. **Auth gate works** (Caddy bearer check)
   - Without token: `curl -sk -o /dev/null -w '%{http_code}\n' -XPOST
     https://metrics.localhost/api/v1/write` → **401**.
   - Wrong path with token → **401** (only `/api/v1/write` is allowed).

3. **Real end-to-end push** (closest to production; preferred over synthetic
   payloads). Build the coord binary and run one role pointed at the receiver:
   - `go build -o bin/coord ./cmd/coord`
   - Run one role with metrics on (any role that starts standalone; `api` or a
     minimal `run` against a throwaway config), e.g.
     `AIOZ_METRICS_URL=https://metrics.localhost/api/v1/write`
     `AIOZ_METRICS_ROLE=api`
     `AIOZ_METRICS_BEARER_TOKEN=<token>`
     `AIOZ_METRICS_INTERVAL=5s` `bin/coord api --config-dir <tmp>` (use a curl
     `--cacert` / `-k` equivalent by pointing at Caddy's internal CA, or run
     the local override that exposes plain-HTTP Prometheus for the test).
   - Wait ~15s, then query Prometheus for a series every role pushes:
     `curl -s 'http://<prometheus>/api/v1/query?query=goroutines\{role="api"\}'`
     → non-empty `result` with `role="api"`. This proves ingest + the `role`
     label round-trips.
   - Alternative if standing up a full coord role locally is heavy: point a
     tiny throwaway `telemetry.NewClient(log, cfg, monkit.Default)` (cfg.URL =
     the receiver, cfg.BearerToken = token) and call one push cycle — reuses
     the exact production client (`pkg/telemetry/client_test.go` shows the
     shape). Then run the same Prometheus query.

4. **Grafana dashboards render**
   - Open `https://grafana.localhost`, log in with the admin password.
   - Datasource **Prometheus** is present, default, and **Test → green**.
   - **Coord** folder contains all three dashboards; with a role pushing, the
     Coord Overview goroutines/memory panels show data.

## Out of scope / notes

- VictoriaMetrics/Cortex/Mimir alternatives, k8s manifests, and worker-side
  push are not part of this change.
- After approval, also write this plan into the Obsidian vault at
  `projects/depin/plans/<date>-monitor-server-hosting.md` and refresh
  `projects/depin/INDEX.md` (per project workflow), via the hermes CLI.
```
