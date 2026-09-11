# aioz-stream: migrate Postgres, Redis, RabbitMQ and outbound HTTP to aioz-common v0.3.0

Status: draft for review (2026-09-10). Not implemented. Branch `refactor/sync-with-template-source`.

## Context

aioz-common v0.3.0 (2026-09-10) ships `postgres`, `redis`, `rabbitmq`, `httpclient`
and `retry`: configured, observable clients with startup ping, health, monkit
stats, and (for rabbitmq) a supervised connection, confirmed publisher and
bounded consumer. aioz-stream still hand-rolls all four:

- `internal/config/postgres.go` - gorm over a DSN string, `TimeZone=Asia/Shanghai`,
  `sslmode=disable`, fixed pool sizes, a stdout gorm logger.
- `internal/config/redis.go` - two go-redis clients (conn-id DB, uuid DB), no
  timeouts, no metrics, never closed.
- `internal/utils/rabbitmq/rabbitmq.go` - streadway/amqp (deprecated), one
  connection per queue, fixed 5s reconnect with no jitter, non-durable queues,
  auto-ack consumers, disk spool of failed publishes.
- ~10 outbound HTTP sites, most with no timeout (`&http.Client{}` per call in
  `internal/utils/stream/stream.go`, `http.DefaultClient` in summarization and
  translate, `resty.New()` for webhooks).

v0.3.0 also carries the uuid `Value()` fix (writes the canonical string), which
the branch is currently missing (pinned `v0.2.1-0.20260908094255-93c696c`).

User decisions (2026-09-10):
- **Config: adopt the common Config structs as config sections** (not a code-side
  mapping of the old keys). Needs a config.yaml migration on every host.
- **Queues:** asked whether common can keep today's declaration. It can -
  `Queue.Durable`, `PublisherConfig.Persistent`, `ConsumerConfig.AutoAck` are plain
  fields and `Topology.Validate` never rejects a non-durable queue.
- **Scope: plan first**, implement after review.

## Plan

### Step 1 - bump aioz-common to v0.3.0
- `go get gitlab.internal/tuan.quang.tran/aioz-common@v0.3.0`. This also brings pgx 5.5.5→5.10, go-redis 9.8→9.22 and amqp091 1.14.
- Fill go.sum with `go test -mod=mod -run '^$' ./cmd/... ./internal/... ./pkg/...`; `go mod tidy` trips on the root-owned `postgres_data/`.
- Brings the uuid `Value()` string fix. Recheck the `internal/tools/uuidprobe` repro and the `payment_logs` insert.

### Step 1b - config: common sections inside today's loader
Current state: `internal/config/config.go` `MustNewAppConfig` uses its own viper, `BindEnv` from `env:` tags (`validate.go` `configBindings`), and `Unmarshal` through mapstructure. No `AIOZ_*` env and no `config.Bind` anywhere; deploy hosts provide only `config.yaml`.

Approach, a hybrid scoped to the four sections, so the rest of `AppConfig` is untouched:
- Replace `PostgresConfig`, `RedisConfig` and `RabbitMQConfig` with `postgres.Config`, `redis.Config` ×2 (`redis_conn_id`, `redis_uuid`), and `rabbitmq.Config`. Tag them `mapstructure:"-"` so viper's Unmarshal skips them.
- Add `HTTP` clients: an `httpclient.Config` per outbound dependency (list in Step 5).
- Defaults:
  - Before decoding, call `config.SetRelease(version.IsRelease())`.
  - Then call `config.BindFlags(pflag.NewFlagSet(...), &section, config.UseReleaseDefaults()/UseDevDefaults())` on a throwaway flagset. This writes the `default`/`devDefault`/`releaseDefault` tags into the struct (the same pattern the template uses in `internal/app/config_test.go:14-22`).
  - Then decode the file's section with `structs.Decode(v.Sub("postgres").AllSettings(), &cfg.Postgres)`. That is hyphen-tolerant, like aioz-config `Load`, and keeps the defaults for keys the file omits.
- Keys follow house style (template `config.example.yaml`): nested yaml with hyphenated leaves, e.g. `postgres.database`, `postgres.ssl-mode`, `postgres.max-conns`, `rabbitmq.address`.
- After loading, call each section's `Validate()` and `config.CheckSecrets` on the four sections. `postgres.password` is `secret:"true"` and has no release default.
- `validate.go` walkers skip foreign-package structs. Extend `Keys()` to list the common sections' keys (derived from the throwaway flagset's flag names), so `TestExampleConfigHasNoUnknownKeys` keeps working.
- Update `env-example/config.yaml` and `env-example/test.yaml`, the root `config.yaml` header (still mentions APP_ENV), `local_yaml_test.go` (the fields are renamed), `internal/seeds/seed.go`, and `internal/cli/migrate` `dsn()`, which becomes `cfg.Postgres.ConnString()`.

**Deploy-affecting defaults to set explicitly in every host's config.yaml:**
- `postgres.ssl-mode: disable`. The release default is `require`, and the current DSN says `disable`.
- `postgres.max-conns: 100` and the pool settings, to preserve today's 100/10/5m/2m.
- `rabbitmq.address`, `rabbitmq.username` and `rabbitmq.password`, replacing `host`/`port`/`default_user`/`default_pass`.

**Ops runbook:** rewrite each host's config.yaml before deploy. The CI migrate-job fails fast when a required secret is missing, via `CheckSecrets`, which goes into the migrate path too. Ship a key-by-key mapping table in the MR description.

Out of scope: moving all of `AppConfig` to aioz-config `Bind`/`Load` (AIOZ_* env, flags). That is a separate migration.

### Step 2 - Redis (small; api only)
- The config section becomes two `redis.Config` values: `redis_conn_id` and `redis_uuid`, one pool per logical DB (1 and 0 today). `Name` tags the metrics.
- `internal/config/redis.go` `MustConnectRedis` is replaced by `redis.Open(ctx, log, cfg)` ×2. Chain `monkit ScopeNamed("redis")`, and `Close` on shutdown (never done today).
- Consumers keep `*goredis.Client`. Pass `client.Client` (it is embedded), so `internal/app/live/media/service.go` (`:733,:798,:805,:809`, only Get/Del/SIsMember) does not change.
- Keys are written by the external mediamtx fork and `deploy/nginx/redis_lookup.lua`. The key layout and DB numbers must not change.

### Step 3 - RabbitMQ (largest)
Queue inventory:

| Queue | Here | Peer | Notes |
|---|---|---|---|
| `callWebhook` | api publishes and consumes | - | Publishers are in `media/service.go` and `hls_import.go`; the consumer is `webhook/consumer.go`. grpc only declares it. |
| `endLiveStream` | api consumes | peer publishes | Handler is `live/stream/service.go:247`. |
| `queue_highlight_assembling` | api publishes and consumes | - | |
| `queue_highlight_manifest_gen` | api publishes and consumes | - | |
| `queue_highlight_detection_task` | api publishes | peer consumes | |
| `queue_highlight_resp` | api consumes | peer publishes | |
| `cdnHandlerCh`, `qualityCh`, `responseCh` | declared only (api and grpc) | ? | Never used; drop them from the wiring. Keep their declaration only if a peer relies on us declaring them (to confirm with the core owners). |

Design:
- **One `rabbitmq.Conn` per process** (today there is one TCP connection per queue, 9 in api). Register it as a lifecycle item with `Run`/`Close`, and chain monkit.
- **Topology** is declared in code, not config: every queue `Durable:false, AutoDelete:false, Exclusive:false`, on the default exchange, with no DLX. This is identical to today, so there is no broker change and no peer coordination. RabbitMQ 4 refuses non-durable non-exclusive queues, so moving to durable queues is a separate, coordinated follow-up (delete the queues, and core, mediamtx and us all declare them durable).
- **Publisher**, one per process: `Exchange:""`, `Persistent:false` (matches today), `Confirms:true`. Each publish passes the queue name as the routing key via `PublishJSON(ctx, queue, v)`, so the payload stays JSON (content-type application/json, as today).
- **Consumers**, one `rabbitmq.Consumer` per consumed queue, replacing `internal/utils/consumer`. Use `AutoAck:false` with manual ack: the handler returns nil to ack; on error the message is nacked without requeue and dropped, since there is no DLX. That matches today's at-most-once outcome but without losing in-flight messages on shutdown (drain). Keep `Prefetch`/`Concurrency` at 1 for the highlight queues, whose handlers rely on DB state-machine claims and ran serially.
- **Disk spool (`WithRecoverData`)**: aioz-common deliberately leaves this to the service. Today it is buggy: `saveMessage` swallows errors, a failed replay duplicates the file, a nil `info` deref, and it republishes messages the caller already treated as failed. It is also unmounted in compose, so it is lost on recreate. Recommend dropping it: `Publish` now blocks until the broker is back (bounded by ctx) and confirms. Callers that already handle publish errors (highlight status rollback) stay correct; media and hls-import callers keep logging. See decision 1.
- **Interfaces**: replace `consumer.Queue` and `highlight.HighlightTaskQueue` (which leak `amqp.Delivery`) with a narrow `Publisher interface{ PublishJSON(ctx, key string, v any) error }` at each consumer site (media, hls_import, highlight). Handlers become `func(ctx, rabbitmq.Delivery) error`, wrapped to decode the body.
- Delete `internal/utils/rabbitmq`, `internal/utils/consumer` and the streadway dependency. Update the fakes in `consumer_test.go`, `highlight/service_test.go` and `webhook/endpoint_test.go`, and `highlight/rabbitmq_integration_test.go`.
- Bonus fixes that come for free: `Consume` called twice registered a broker consumer and then errored; the reconnect goroutine died after one replay failure; grpc had no `depends_on` rabbitmq.

### Step 4 - Postgres: GORM on the common pool
- `internal/config/postgres.go` `MustConnectPostgres` → `postgres.OpenWith(ctx, log, poolCfg, cfg.Postgres)`, where `poolCfg, _ := cfg.Postgres.PoolConfig()` and then `poolCfg.ConnConfig.RuntimeParams["timezone"] = "Asia/Shanghai"`. The common package never sets TimeZone. Three queries depend on the session zone and would silently shift day and month boundaries without it: `usage.repository.go:307` (`date_trunc('day', ...)`), `payment.repository.go:142-165` (timestamp-without-zone vs `time.Time` params), and `user.repository.go:76` (`now() - interval '1 month'`).
- GORM over the pool: `sqlDB := stdlib.OpenDBFromPool(db.Pool)`; `gorm.Open(gormpg.New(gormpg.Config{Conn: sqlDB}), &gorm.Config{Logger: ...})`. Every repository, the payment lib (which takes `*gorm.DB`) and cross-service transactions stay unchanged. The pgxpool owns the limits, so leave the `database/sql` limits unset.
- Keep `captureQueryOnError`. Replace the stdout gorm logger with a slog-backed one (warn and slow queries >1s, ignore RecordNotFound, `ParameterizedQueries: true`; today the flag is `false`, which inlines values into logs, contrary to its comment). Panics carry the real `err` instead of fixed strings.
- Lifecycle: `monkit.ScopeNamed("postgres").Chain(db)`; close the pool at shutdown in api and grpc.
- Migrate (`internal/cli/migrate/migrate.go` `dsn()`) → `cfg.Postgres.ConnString()`, plus `CheckSecrets`.
- `internal/seeds/seed.go` follows the same path.

### Step 5 - outbound HTTP via `httpclient`
Add one `httpclient.Config` section per dependency under `http:` (e.g. `http.stream`, `http.cdn`, `http.transcribe`, `http.summary`, `http.translate`, `http.ip2location`, `http.webhook`, `http.graylog`, `http.slack`, `http.mail`). Build each once in wiring and inject it. Retries stay off, except where noted.

| Site | Change | Retry | Notes |
|---|---|---|---|
| `internal/utils/stream/stream.go` (8 per-call `&http.Client{}`, no timeout) | inject one client into `NewStreamClient` | `WithRetry` on the GETs/DELETEs; not on the create/kick POSTs | Drop the hand-rolled backoff loop in `GetStreamPathWithStreamType` for `WithRetry` |
| `internal/utils/storage/cdn.go` (new transport per call) | inject a pooled client; keep per-request size-based deadlines via ctx | keep the lib retry off; keep the existing rewind logic only for multipart; fix `Transcode` resending a consumed body | Raise `MaxResponseSize` to 0 (off) for this client: `Download` streams large bodies |
| transcribe, summarization (`http.DefaultClient`), ip2location | inject clients | `WithRetry` on GETs only (`GetTaskResult`, `GetTaskDetail`, ip2location) | Fix the transcribe body leak on non-200 |
| translate (`wire.go:576` passes `http.DefaultClient`) | pass the configured client | no | |
| graylog (`&http.Client{}`, one goroutine per record, no timeout) | configured client with a short timeout (e.g. 5s) | no | Unbounded goroutines on a hung Graylog are the real bug; a timeout bounds them |
| Slack `slack.OptionHTTPClient`, Resend `resend.NewCustomClient` | inject clients | no | |
| webhook (`resty.New()`, user URLs) | `resty.NewWithClient(client)` or plain net/http | no (DB retry chore already) | See decision 2 (SSRF) |
| hlsimport (SSRF dialer and per-hop `CheckURL`) | `httpclient.NewWith(log, cfg, base)`, where `base` is its transport with the `Control` dialer; then re-assign `CheckRedirect` on the returned client, because `NewWith` overwrites it | `WithRetry` on GETs | `MaxResponseSize: 0` (segments up to 200 MiB are capped by its own limits) |

Out of reach: the payment lib's own `http.Get` / cosmos / eth clients (needs a lib change).

### Step 6 - cleanup
Remove streadway/amqp, resty (if webhook moves to net/http) and the dead queue fields. Update `docker-compose.yml`: grpc gets `depends_on: rabbitmq`; redis loads its mounted conf.

## Decisions to confirm (flagged, defaults chosen)
1. **Disk spool:** drop it (default), since the confirmed, blocking publisher replaces it. The alternative is to port it onto `Publisher` as a service-side policy.
2. **Webhook SSRF:** today there is none (it accepts `127.0.0.1`, `169.254.169.254`, internal names). Default: in this migration, reuse the hlsimport dial-time guard (`checkDialAddress`/`IsPublicIP`, moved to a shared package) as the webhook client's `base` transport. Otherwise a separate follow-up.
3. **TimeZone:** keep `Asia/Shanghai` for now. Separately, the containers run with `TZ=Asia/Ho_Chi_Minh` (UTC+7) against a UTC+8 DB session, which is worth a follow-up.
4. **Durable queues:** a follow-up, coordinated with the core and mediamtx owners. RabbitMQ 4 forces it.

## Order and commits
One commit per step, each green before the next: 1 (bump) → 1b (config) → 2 (redis) → 4 (postgres) → 5 (http) → 3 (rabbitmq) → 6 (cleanup).

## Verification
- Local stack: `aioz-stream-db` :5437, redis, rabbitmq 3 on :5675. Boot api and grpc with a CDN stub (the `/getBalance` stub recipe in memory `single-binary-cli`).
- **Config:** extend `config_yaml_test` / `config_test` for the new sections (defaults applied, release `ssl-mode` = `require` unless set, `CheckSecrets` fails on an empty postgres password). Load `env-example/config.yaml` in dev and release channels.
- **Postgres:**
  - api/grpc boot; `SHOW timezone` over the app pool returns `Asia/Shanghai`.
  - The integration tests (`MIGRATE_TEST_DSN`), including `TestModelsMatchTheMigratedSchema`.
  - The `uuidprobe` repro and a `payment_logs` insert.
  - monkit `postgres` series on `:6060/metrics`.
- **Redis:** `/disconnect` hook and `SweepMediasNotStreaming` against seeded keys (`SET <uuid> server1`, `SADD server1 <connId>`), then check the Del effects. `redis` series in metrics.
- **RabbitMQ:**
  - `highlight/rabbitmq_integration_test.go` against the local broker.
  - E2E: upload-complete publishes `callWebhook`, which the consumer delivers to a local HTTP sink.
  - Publish `endLiveStream` by hand and see the stream end.
  - Restart the broker mid-run: publishes pause and resume, consumers re-register, and there is 1 connection per process (management UI).
  - `rabbitmqctl list_queues name durable` is unchanged.
- **HTTP:** point stream, transcribe and CDN at local stubs that hang, and confirm the configured timeout fires. hlsimport keeps rejecting `127.0.0.1` and redirect hops to disallowed hosts (existing fetcher tests). If decision 2 is in, webhook delivery to `127.0.0.1` is refused.
- `make lint`, `make test` (race) and swagger-verify all green; `go test -tags integration ./internal/migrations/`.

## Revision 2026-09-11 - implemented at aioz-common 6b79f13 (with SSRF guard)

Status: implemented and verified on branch `refactor/sync-with-template-source`, uncommitted. Pin: `v0.2.1-0.20260910155547-6b79f1327b30` - commit 6b79f13 on aioz-common `feat/http-postgres-retry`, which is v0.3.0 plus the httpclient private-network guard. It is untagged; re-pin to v0.3.1 once tagged.

Decisions made during implementation (user, 2026-09-11):
- Webhook SSRF is in scope: `httpclient.New` with `private-networks: block` by default, and a release build refuses `allow`. `Check` returns 422 `webhook-url-not-public`; `Notify` does not queue a retry for a blocked address.
- hlsimport uses the common guard (`httpclient.Transport` + `NewWith`, with `CheckRedirect` re-set for `CheckURL`). `IsPublicIP` is removed, and `ErrBlockedAddress` maps to `ErrPrivateAddress`.
- `GetStreamPathWithStreamType` keeps its own retry loop, because it also retries a 200 with an empty path. The other stream GETs and DELETEs use `WithRetry`.
- `cdnHandlerCh`, `qualityCh` and `responseCh` are dropped entirely.
- `deploy/redis/redis.conf` now uses `bind * -::*` and `protected-mode no`, and is loaded through the compose `command:`.

Deviations from the plan text:
- The migrate `dsn()` items are dropped; that package was removed in 2f147c49.
- Per-client http defaults: cdn has timeout 0 and max-response-size 0, graylog 5s, transcribe and ip2location 3s, translate 15s.
- Every consumer runs with Prefetch and Concurrency 1, which keeps today's serial handling.
- The rabbitmq connection is named `aioz-stream`.
- An unknown key in a common section stops startup.

Verification:
- `make lint` reports 0 issues; `make test` (race) and `swagger-verify` pass.
- The migrations integration suite passes on a scratch database.
- Postgres integration: the session timezone is Asia/Shanghai, and the query log keeps bound values out.
- The webhook guard tests pass through the HTTP endpoint; the hlsimport fetcher tests pass.
- The hung-server timeout tests pass for stream and transcribe.
- Broker E2E on a throwaway RabbitMQ: webhook publish, consume and deliver, and the highlight assembling round trip.
- Booted api and grpc:
  - 2 broker connections (one per process; the old api alone opened 9) and 5 consumers.
  - The non-durable topology is unchanged.
  - After a broker restart, reconnection took 16s.
  - Metrics cover postgres, redis, rabbitmq and 10 http clients.
  - SIGTERM exits in 5s with code 0, leaving 0 connections.

Not run (they need live-stream data): the redis `/disconnect` + `SweepMediasNotStreaming` E2E, publishing endLiveStream by hand, and the upload-complete path.

Ops: rewrite each host's config.yaml before deploying (`postgres.database`, `ssl-mode` and the pool, `redis_conn_id`/`redis_uuid`, `rabbitmq.address`/`username`/`password`, `http.webhook`). `storage.recover_path` is now unused.

### Addendum 2026-09-11 - re-pinned to aioz-common 5343511

The pin moved from 6b79f13 to `5343511` ("refactor(httpclient): move retry from transport to caller-driven policy"): `v0.2.1-0.20260911043352-53435114951d`. It is still untagged; re-pin to v0.3.1 once tagged. The SSRF guard is unchanged.

What 5343511 changes:
- `WithRetry`/`WithoutRetry` and the `Retry*` fields in `httpclient.Config` are removed, and the client never retries on its own.
- A call site retries explicitly with `httpclient.Retry{...}.Do(ctx, client, newRequest)`.

Migration in aioz-stream:
- The stream, transcribe, summarization, ip2location and hlsimport clients each hold a `retry` policy.
- The GET/DELETE sites call `retry.Do` with a request builder, so headers are set on every attempt. The stream client gets a small `request(method, target)` builder.
- hlsimport's `Retryable` does not retry its own refusals (host not allowed, too many hops, bad scheme), because `Retry.Do` now sees redirect-policy errors.
- Each attempt now has its own client timeout, and the caller's context deadline bounds the whole retry sequence.

Verified:
- The hung-server timeout tests, now with per-case caller deadlines.
- A new test: a GET is retried after a 503, and a POST is sent once.
- The hlsimport fetcher tests.
- Every test package compiles.
