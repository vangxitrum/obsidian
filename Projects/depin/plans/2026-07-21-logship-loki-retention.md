# Log shipping (Loki) + metrics retention for the test fleet

## Context

While debugging why the relay's Grafana "connected peers" count didn't match
through-relay ping results, we hit a wall: the relay's own logs (which turned
out to hold the actual root-cause evidence - a burst of mass disconnect/
reconnect across two shared test-fleet IPs) weren't centrally collected, and
neither are worker logs. The existing `monitoring/` stack only handles metrics
(Prometheus remote_write + Grafana), and CPU/memory-per-process telemetry
already works for `coord` roles today but isn't turned on everywhere.

Goal of this change: add centralized log collection (via Grafana Loki, fitting
the existing Prometheus+Grafana+Caddy stack) for both `worker` and `coord`
processes, with the log-shipping destination delivered as a **hidden**
(non-CLI-editable) config value baked into deployments, and set sane retention
(15 days, sized per-engine toward a combined ~100GB budget) across both
Prometheus and Loki.

**Explicitly confirmed decisions (do not re-litigate):**
- Log shipping is a **custom Go push client** (mirrors the existing metrics
  telemetry client), not a Promtail/Vector sidecar.
- Retention is sized **natively per engine** (Prometheus supports both a time
  and a size flag; Loki is primarily time-based, its size budget is enforced
  by disk provisioning, not a hard flag).
- Coord CPU/memory collection is **out of scope for new code** - it already
  works today via the existing `pkg/telemetry` client the moment
  `--metrics.url` is set for a given coord role. This plan does not touch that.
- **Network usage metrics and the "which machine" worker tag are explicitly
  deferred** - not part of this plan.

---

## Part 1: `pkg/logship` - the Go log-push client

New package, mirroring `pkg/telemetry`'s shape and lifecycle conventions
(`pkg/telemetry/client.go`, `pkg/telemetry/config.go`).

```
pkg/logship/
  config.go        // Config struct, Enabled()
  client.go         // Client: ring buffer, HTTP push (Loki JSON), Run/Close (lifecycle.Item)
  core.go           // Core: zapcore.Core wrapping *Client
  client_test.go
  core_test.go
```

### `config.go`

```go
type Config struct {
	// Hidden: never a CLI flag (never in --help, never --flag=value,
	// never written by `setup`/SaveConfig). Only reachable via a value
	// literally present in config.yaml/secrets.yaml - see Part 3 for why.
	URL         string `noflag:"true" help:"Loki push endpoint (POST .../loki/api/v1/push); empty disables log shipping" default:""`
	BearerToken string `noflag:"true" user:"true" help:"bearer token for the Loki push endpoint" default:""`

	App      string `help:"app label attached to every shipped log stream" default:"coord"`
	Role     string `help:"role label attached to every shipped log stream" default:""`
	Instance string `help:"instance label attached to every shipped log stream; empty resolves to hostname at startup" default:""`

	// Independent of the process's own --log.level; defaults to "info" so a
	// local --log.level=debug session doesn't flood the shared Loki.
	Level string `help:"minimum level shipped to Loki: debug/info/warn/error/dpanic/panic/fatal" default:"info"`

	FlushInterval    time.Duration `help:"how often buffered log lines are pushed to the Loki endpoint" default:"5s"`
	MaxBufferedLines int           `help:"max buffered log lines held in memory; oldest dropped first once full" default:"10000"`
	MaxLineBytes     int           `help:"truncate a single formatted log line to this many bytes before buffering (0=unlimited)" default:"65536"`
}

func (cfg Config) Enabled() bool { return cfg.URL != "" }
```

`Level` is a plain string (not `zapcore.Level`) because cfgstruct's flag
registration doesn't special-case that type.

### `client.go`

- `Client` holds a fixed-capacity **ring buffer** (`MaxBufferedLines`) written
  to by `Core.Write` and drained by a `sync2.Cycle`-driven `Run(ctx)` (same
  periodic-push pattern as `telemetry.Client.Run`) every `FlushInterval`.
- **Backpressure policy: drop-oldest.** When the buffer is full, the oldest
  line is overwritten. Rationale: when Loki is down, the most useful signal
  for live debugging is "what's happening now," not a growing backlog of
  stale history; this also self-heals the instant the push loop catches up.
  Mirrors telemetry's own "drop a failed interval's data, don't retry" policy.
- `flush()` drains the buffer, stable-sorts by timestamp (defensive - Loki
  requires non-decreasing timestamps per stream per push), and pushes in
  batches of `maxLinesPerPush = 5000` via `errs.Group` (partial-failure
  tolerant, like `telemetry.Client.report`).
- **Push format**: plain JSON to Loki's push API (not the protobuf/snappy
  remote_write format the metrics client uses - simpler, no new vendored
  proto needed, adequate for this fleet's volume):
  ```json
  {"streams":[{"stream":{"role":"...","instance":"...","app":"..."},"values":[["<unix_nano>","<line>"],...]}]}
  ```
- Auth: `Authorization: Bearer <BearerToken>` only (mirrors
  `telemetry.Client.setAuth`'s bearer path; no basic-auth fallback needed
  here).
- `Instance` resolution (hostname, falling back to parsing `/proc/self/cgroup`
  for a container ID) - duplicate the existing unexported
  `resolveInstanceID`/`containerID`/`parseContainerID` helpers from
  `pkg/telemetry/client.go` rather than extracting a shared package (keeps
  the diff to the stable, already-tested telemetry package at zero).
- `Run`/`Close` satisfy `lifecycle.Item{Name, Run, Close}` exactly like the
  telemetry client. `Close` performs one bounded (5s timeout) best-effort
  final flush.
- **Critical invariant**: the `*zap.Logger` passed into `NewClient` (used to
  warn about push failures) must be the **pre-wrap** logger - i.e. captured
  *before* `zap.WrapCore` attaches the Loki `Core` - so a push failure's own
  warning log can never feed back into the very buffer that just failed to
  flush. This falls out naturally from the wiring order in Part 2.

### `core.go`

- `Core` implements `zapcore.Core`: `Enabled`/`With`/`Check` standard zap
  patterns; `Write` renders the entry via an internal JSON encoder
  (independent of the process's local `--log.encoding`, no ANSI color codes)
  and calls `client.enqueue(...)` - **no I/O**, O(1), never blocks.
- `Sync()` is a **no-op** - it must never perform a synchronous network push
  (zap calls `Sync()` on every Error+ level entry on the hot path). Buffered
  lines are flushed only by `Client.Run`'s timer and `Client.Close`'s final
  flush.

### Testing (`client_test.go`, `core_test.go`)

Mirror `pkg/telemetry/client_test.go`'s conventions (`httptest.Server` capture,
`testcontext.New(t)`, `zaptest.NewLogger(t)`). Cover: push happy path + label
correctness, timestamp sorting, drop-oldest-when-full + dropped-count,
line truncation, chunking across `maxLinesPerPush`, push-failure-is-warned-
and-dropped-not-retried, bearer auth header, `Run`/`Close` lifecycle shape,
disabled-client no-ops, and **an explicit anti-recursion regression test**:
wrap a real logger via `zap.WrapCore` exactly as production wiring does,
point it at an always-500 test server, log through the wrapped logger, and
assert the *next* flush's payload does not itself contain a "logship push
failed" line.

---

## Part 2: Wiring into `coord` and `worker`

Attach the `Core` via `zap.WrapCore(func(core zapcore.Core) zapcore.Core {
return zapcore.NewTee(core, logshipCore) })` at the **earliest possible point**
in each constructor - not next to the existing `setupTelemetry`/relay
telemetry block, because tracing the actual code found subsystems in all
three constructors that capture the logger *before* that point:

- **`worker/peer.go`, top of `NewPeer`** (before `peer := Peer{Log: log, ...}`
  is built) - add a new `LogShip logship.Config` field to `worker.Config`
  (`worker/peer.go:49-75`, next to `Debug`). If `config.LogShip.Enabled()`,
  build the `logship.Client` from the pre-wrap `log`, reassign `log` via
  `WrapCore`, and add the client as the **first** item in `peer.Servers`
  (`lifecycle.Group` closes in reverse order, so this closes *last*,
  catching shutdown-time logs from everything else).
- **`coord/peer.go`, top of `newBase`** (`coord/peer.go:104-118`) - same
  pattern. This is earlier than `setupServer()`/`setupDebug()`, both of which
  run before `setupTelemetry()` and would otherwise miss the wrap. Add
  `LogShip logship.Config` to `coord.Config` (`coord/config.go:36`, next to
  `Metrics`).
- **`coord/relay/peer.go`, top of `NewPeer`** - earlier than the existing
  inline telemetry block (~line 212) and earlier than `tracer := &metricsTracer{...}`.
  Add `LogShip logship.Config` to `coord/relay/config.go`'s `Config`
  (next to `Metrics`).

Existing `setupTelemetry()` and the relay's inline telemetry block are left
completely untouched.

---

## Part 3: Delivering the hidden URL/token (fleet packaging)

**Definitive finding** (independently confirmed twice, by tracing
`internal/cfgstruct/cfgstruct.go`, `pkg/process/exec_conf.go`, and the
vendored `github.com/spf13/viper` source): a `noflag:"true"` field is *never*
registered as a pflag, so it can never enter Viper's `AllSettings()` via
`AutomaticEnv` - env vars like `AIOZ_WORKER_LOGSHIP_URL` **will not** reach it,
no matter what. The only path in is a key literally present in
`config.yaml`/`secrets.yaml` (merged via `LoadConfig`'s `ReadInConfig`/
`MergeInConfig`, entirely independent of flag binding).

This means the fleet's current runtime-override mechanism (`entrypoint.sh`
exporting `AIOZ_WORKER_*` env vars) cannot deliver this value - a different,
file-based mechanism is needed:

1. **`dev/fleet/package-worker.sh`**: add `FLEET_LOGSHIP_URL` /
   `FLEET_LOGSHIP_BEARER_TOKEN` (sourced from `control.defaults.env`,
   overridable via new `--logship-url`/`--logship-token` flags - same pattern
   as the existing `--control-host`), passed as two more `--build-arg`s.
2. **`dev/fleet/Dockerfile.fleet`**: add matching `ARG`/`ENV` lines, mirroring
   the existing `FLEET_CONTROL_HOST` baking exactly.
3. **`dev/fleet/entrypoint.sh`**: write a `secrets.yaml` into `$CONFIG_DIR`
   **unconditionally on every container start** (not gated like the one-time
   `config.yaml` write - there's no "don't clobber operator edits" concern
   here):
   ```bash
   LOGSHIP_URL="${LOGSHIP_URL:-${FLEET_LOGSHIP_URL:-}}"
   LOGSHIP_BEARER_TOKEN="${LOGSHIP_BEARER_TOKEN:-${FLEET_LOGSHIP_BEARER_TOKEN:-}}"
   if [ -n "$LOGSHIP_URL" ]; then
     cat > "$CONFIG_DIR/secrets.yaml" <<EOF
   logship:
     url: "$LOGSHIP_URL"
     bearer-token: "$LOGSHIP_BEARER_TOKEN"
   EOF
   fi
   ```
   (`zeebo/structs`' key matching is case-insensitive with hyphens stripped,
   so `bearer-token` correctly maps to the `BearerToken` Go field.)

Known, accepted tradeoff: baking the bearer token as a Docker image `ENV` is a
minor secret-hygiene weakening (same tradeoff already accepted for
`FLEET_COORD_TRUST_SOURCE`) - acceptable for this test fleet, flagged as a
possible future hardening (template `secrets.yaml` into the shipped zip
instead of the image layer).

---

## Part 4: Monitoring stack (Loki service, retention, Caddy, Grafana)

### `monitoring/docker-compose.yml` - new `loki` service

```yaml
  loki:
    image: grafana/loki:3.6.12
    restart: unless-stopped
    command:
      - -config.file=/etc/loki/loki-config.yaml
      - -config.expand-env=true
    environment:
      LOKI_RETENTION: ${LOKI_RETENTION:-360h}
    volumes:
      - ./loki/loki-config.yaml:/etc/loki/loki-config.yaml:ro
      - loki-data:/loki
```
Add `loki-data:` to the top-level `volumes:`; add `loki` to `caddy`'s
`depends_on`; add `LOKI_BEARER_TOKEN: ${LOKI_BEARER_TOKEN:?set LOKI_BEARER_TOKEN in .env}`
to `caddy`'s `environment:`. Not published to the host - reached only via Caddy,
matching the Prometheus pattern.

### `monitoring/loki/loki-config.yaml` (new)

Single-binary mode, filesystem storage (no S3/GCS - matches this stack's
scale), `schema: v13`/`tsdb` index, `limits_config.retention_period:
${LOKI_RETENTION:360h}`, `compactor.retention_enabled: true` with
`delete_request_store: filesystem`, `analytics.reporting_enabled: false`.
(Full YAML drafted during the design pass - write it verbatim at
implementation time.)

### Retention split: Prometheus 25GB/15d, Loki ~75GB/15d

- **Prometheus** (`docker-compose.yml`, `prometheus` service `command:`):
  flip `--storage.tsdb.retention.time` default from `30d` to `15d`, add
  `--storage.tsdb.retention.size=${PROM_RETENTION_SIZE:-25GB}` (a real,
  already-supported v2.53.0 flag, simply unused today). Reasoning: Prometheus
  ingestion here comes from a small, bounded set of `coord`/`edge` processes
  (~10-20 instances) - 25GB is ~10x the estimated real footprint.
- **Loki**: `retention_period: 360h` (15d) enforced by the compactor: no
  native total-bytes lever exists, so the ~75GB share is a **disk
  provisioning target**, not a hard cap - size the `loki-data` volume/disk to
  ~80GB and treat it as an alerting threshold. Reasoning: Loki's ingestion is
  driven by the ~95-worker fleet (the reason this whole feature exists),
  dwarfing Prometheus's process count; back-of-envelope at ~2 lines/sec/worker
  and ~10x Loki's typical compression ratio lands around 4-8GB over 15 days -
  75GB leaves generous headroom, to be revisited once real volume is observed.

### Caddy (`monitoring/caddy/Caddyfile`) - new route, separate token

```caddyfile
	@logpush {
		path /loki/api/v1/push
		header Authorization "Bearer {$LOKI_BEARER_TOKEN}"
	}
	handle @logpush {
		reverse_proxy loki:3100
	}
```
Added inside the existing `{$METRICS_DOMAIN} { ... }` block, before the
catch-all `handle { respond "Unauthorized" 401 }`. **Deliberately a separate
token from `METRICS_BEARER_TOKEN`**: the worker-fleet image (carrying the
baked log-ship token) reaches far more machines than coord's metrics token
ever does, so a leaked/compromised worker image must not also grant
Prometheus remote-write access.

### Grafana

- `monitoring/grafana/provisioning/datasources/loki.yml` (new): registers
  Loki, `isDefault: false` (Prometheus stays default so existing dashboards'
  `${DS_PROMETHEUS}` variable is unaffected).
- `monitoring/grafana/dashboards/logs/` (new folder) + a new `providers:`
  entry in `monitoring/grafana/provisioning/dashboards/dashboards.yml`
  (`folder: Logs`, `path: /var/lib/grafana/dashboards/logs`). Starter
  dashboard: `app`/`role`/`instance` template variables (Loki
  `label_values(...)`), a log-volume-over-time panel
  (`sum by (role) (count_over_time({app=~"$app", role=~"$role"}[$__interval]))`),
  and a scrollable logs panel with a free-text search filter.

### `.env.example`

Add `LOKI_BEARER_TOKEN`, `PROM_RETENTION=15d`, `PROM_RETENTION_SIZE=25GB`,
`LOKI_RETENTION=360h`.

---

## Verification

1. **Go code**: `go build ./...` and `go test ./pkg/logship/...` green,
   including the anti-recursion regression test. `go vet` clean.
2. **Wiring smoke test**: run `worker api` (or a coord role) locally with
   `LogShip.URL` pointed at a throwaway `httptest`-style local HTTP listener
   (or `nc -l`) and confirm POSTed JSON bodies appear with correct
   `role`/`instance`/`app` labels and rendered log content.
3. **Fleet packaging**: after the `entrypoint.sh`/`Dockerfile.fleet`/
   `package-worker.sh` changes, run `dev/fleet/package-worker.sh
   --logship-url=... --logship-token=...`, start the resulting image, and
   `docker exec <container> cat $CONFIG_DIR/secrets.yaml` to confirm the
   `logship:` block is present and correctly keyed.
4. **Monitoring stack**: `docker compose up -d` in `monitoring/`, then:
   - `curl -sk -o /dev/null -w '%{http_code}\n' -XPOST https://$METRICS_DOMAIN/loki/api/v1/push` (no token) → expect `401`.
   - Same with `Authorization: Bearer $LOKI_BEARER_TOKEN` and a minimal valid
     Loki push JSON body → expect `204`.
   - Same with `METRICS_BEARER_TOKEN` instead → expect `401` (proves token
     separation actually works).
   - `docker exec <loki-container> wget -qO- http://localhost:3100/config`
     and confirm `retention_period: 360h` / `retention_enabled: true` appear
     in the dumped merged config.
   - Open Grafana's new "Logs" folder/dashboard and confirm log lines from a
     locally-run worker/coord process (step 2 or 3) show up, filterable by
     `role`/`instance`.
5. **Retention, longer-horizon check**: `docker exec <prometheus-container>
   curl -s http://127.0.0.1:9090/api/v1/status/tsdb` for head/block stats;
   `du -sh` on both `prometheus-data` and `loki-data` volumes at intervals
   over the following days to confirm growth flattens near the 15-day mark
   rather than growing unboundedly.

---

## Explicitly out of scope (per earlier decisions)

- Network usage metrics (CPU/memory already covered by existing
  `pkg/telemetry`; network would need new per-process collection code).
- The "which machine" worker/coord tag - dropped for this round.
- Loki multi-tenancy (`X-Scope-OrgID`) and gzip-compressed push payloads -
  noted as easy follow-ups if ever needed, not required to ship this.
