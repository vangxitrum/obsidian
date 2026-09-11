---
name: local-dev-env
description: How to run aioz-stream API locally - env files, infra containers, and the one remaining Slack startup blocker
metadata:
  type: project
---

Set up local dev environment for the `api` binary (`cmd/http`), 2026-07-09.

**Config loading:** `cmd/http/init.go` reads `./{APP_ENV}.env` (default `APP_ENV=debug` →
`./debug.env` in repo root), NOT `app.env` directly — `app.env` is only what docker-compose's
`env_file:` mounts for the containerized `api`/`postgres`/`rabbitmq`/`redis` services. Both
`app.env` (docker-internal hostnames: `postgres`/`redis`/`rabbitmq`) and `debug.env` (host-side:
`localhost` + docker-mapped ports 5437/6379/5675) now exist at repo root, gitignored, generated
with fresh RSA keypairs (JWT access/refresh, since despite the "eddsa" naming in
`env-example/app.env` the code in `internal/utils/token/token.go` actually does
`jwt.ParseRSAPrivateKeyFromPEM` + `RS256` — must be base64(PEM), not raw EdDSA) and random
dev Postgres/RabbitMQ passwords. Redis password is NOT free to choose — it's pinned by the
committed `redis.conf`'s `requirepass` line (`w8EnWLByKJaDht2kLaiNdq2BGRo4CgaSRpynKgjfCfXjBFP471`),
must match in both env files.

**Repo gap found:** `Dockerfile.postgres` does `COPY ./00_init.sql` expecting it at repo root,
but it only lives under `env-example/00_init.sql` — docker build fails until copied to root.
Copied it there (untracked/gitignored copy) to unblock `docker compose up`.

**Missing config keys:** `internal/config/config.go`'s `AppConfig` has `LibreTranslateBaseUrl`,
`SummaryBaseUrl`/`ApiKey`/`ModelId` fields with no corresponding entries in
`env-example/app.env` at all. `translateclient.MustNewLibreTranslateTranslator` panics if
`LIBRE_TRANSLATE_BASE_URL` is empty — added placeholder non-empty values for these to both env
files (no eager network call at construction, so placeholders are fine to unblock boot).

**Infra:** `docker compose --profile vod up -d postgres redis rabbitmq` — all three reach healthy.
`go build -tags purego -o bin/api ./cmd/http/.` (== `make build`) compiles clean. Go 1.25 toolchain,
`aioz-depin/go-sdk` local replace path, and module cache for the private `10.0.0.50/*` modules were
all already in place (see [[gosdk-storagehelper]]).

**Remaining hard blocker (confirmed by actually running the binary):** `cmd/http/init.go:463`
calls `message.NewSlackHelper(OAUTH_TOKEN_BOT_REPORT_CONTENT, CHANNEL_REPORT_COTENT_ID)`, which
does an eager `slackClient.AuthTest()` — panics with `invalid_auth` on any placeholder token.
This is the ONLY thing standing between `APP_ENV=debug ./bin/api` and a full boot (Postgres,
Redis, RabbitMQ, db-trigger exec, payment-gateway RPC dial to mainnet all passed clean in the
live run). User was asked real-token vs. code-detour vs. leave-blocked; chose **leave it
blocked for now** — so a real Slack bot OAuth token for the report-content bot is still needed
before the API will actually start locally.

**How to apply:** next session working on this repo, don't re-derive the above — just get a
real `OAUTH_TOKEN_BOT_REPORT_CONTENT` (+ matching `CHANNEL_REPORT_COTENT_ID`) into
`debug.env`/`app.env` and `APP_ENV=debug ./bin/api` (or `make run` / `air` for hot reload) should
boot clean.

**2026-07-24 wiped local dev Postgres (`./postgres_data`, container `aioz-stream-db`).**
User asked to "clear our old data" after the go-sdk registration work above — confirmed
scope via AskUserQuestion: full wipe, dev DB only (not truncate-in-place). `./postgres_data`
is root-owned on the host (postgres image runs as uid 999 inside), so a plain host `rm -rf`
403'd — used a throwaway `docker run --rm -v ./postgres_data:/data alpine rm -rf /data/*`
instead of `sudo`, then `docker compose stop postgres && rm -f` + `docker compose --profile
vod up -d postgres` to recreate. Confirmed empty via `\l` — only `postgres`/`template0/1`/
`video-db` (the latter re-created empty by `Dockerfile.postgres`'s `00_init.sql`, not old
data). No app tables until something runs `AutoMigrate` (i.e. `api` boots) — `api` container
itself was not running at the time (only `fe`/`db`/`redis`/`rabbitmq` were up).


**Update 2026-09-08 (`~/work/stream/aioz-stream`, branch `feat/new-cdn`).** That
checkout is an OLDER state than the refactor worktree: still env config, not
YAML, and files like `Dockerfile.api` and `00_init.sql` at the repo root. A
YAML config would be wrong there.

Its `debug.env` was regenerated: now 85/85 AppConfig keys, backed up first as
`debug.env.bak.<timestamp>`. What changed:
- **`RECOVER_PATH` was missing entirely.** `rabbitmq.WithRecoverData("")` does
  `filepath.Join("", queueName)`, so it created a directory named after each
  queue **in the repo root** - that is what the stray `callWebhook/`,
  `cdnHandlerCh/` and `endLiveStream/` directories are. Now `./recover`.
- rate limits set to the models defaults (100/200/500) so local matches prod
  instead of drifting to whatever is compiled in.
- `RTMP_URL` / `LIVE_SERVER_URL` given local values: empty produced malformed
  live-stream URLs rather than an error, which is worse to debug.
- The 7 keys AppConfig never reads (`HLS_PORT`, `ZIP_SIZE`, `LOKI_*`,
  `STREAM_HLS_URL`, `GF_SECURITY_ADMIN_*`) are commented out with a note - they
  belong to the compose containers, not the service.

Seven keys are deliberately still empty (`BETTER_STACK_TOKEN`, `SUMMARY_API_KEY`,
`SUMMARY_MODEL_ID`, `GRAYLOG_HOST`, `GRAYLOG_PORT`, `DEMO_VIDEO_ID`,
`LIBRE_TRANSLATE_API_KEY`) - every one is guarded by a `!= ""` check, so empty
disables the feature rather than breaking boot.

**The Slack blocker is already worked around in that checkout**: `cmd/http/init.go`
has the `panic("Could not init Message helper")` commented out as an
uncommitted local change. So the API boots without a real bot token - but that
edit is local only, and anyone else cloning still hits the panic.


**`debug.yaml` in the refactor worktree (2026-09-08).** After the config moved
to YAML, `cmd/http` reads `./<APP_ENV>.yaml`, so local dev needs a
`debug.yaml`, not `debug.env`. Generated by mapping the working values from
`~/work/stream/aioz-stream/debug.env` through the new grouped schema (76 of 94
keys set; the rest are commented out with their env name beside them, and every
one of those is guarded by a `!= ""` check).

**The config switch had left a hole in `.gitignore`.** It covers `*.env` but
nothing for the new YAML, so `/app.yaml` and `/debug.yaml` at the repo root -
carrying the database password and the JWT private keys - would have been
committed. Now ignored, anchored to the root (`/debug.yaml`, not `debug.yaml`)
so a legitimately tracked YAML elsewhere is unaffected, with
`!env-example/*.yaml` keeping the reference copies in git.

**Zero is a restrictive limit, not an unlimited one.** `hls_import.enabled:
true` with the limits left unset rejects every import - `resp.ContentLength > 0`
and `len(segments) > 0` both fire - failing with "limit is 0" rather than
working. The limits are set from env-example/app.yaml;
`allow_private_hosts: true` is the one deliberate local-only difference, since
a dev source is usually on localhost.

`internal/config/debug_yaml_test.go` loads the file through the real
`MustNewAppConfig` and skips when it is absent, so it verifies a developer's
own config without failing anyone else's checkout.


**The Slack boot blocker is fixed properly (2026-09-08), not worked around.**
`message.NewSlackHelper` returned an error that `cmd/http/init.go` turned into
a panic, so a missing or expired bot token stopped the whole API from serving -
over a feature that posts content reports to a channel. That is why the other
checkout had the panic commented out as an uncommitted local edit.

Now `message.New(log, message.Config{Enabled, Token, ChannelID})`, and it
**never fails**. Three paths, each returning a working `MessageHelper`:
- `Enabled: false` -> info log, no-op helper;
- enabled with no token or channel -> **warn**, no-op helper;
- enabled but `AuthTest` fails -> **warn**, no-op helper. The API serves, and
  an operator who reads the startup log fixes the token without a rollback.

**Returning a no-op rather than nil is the point**: `report.Service` holds a
`message.MessageHelper` and every call site would otherwise need its own nil
check. The no-op still logs the report it would have posted, so turning Slack
off does not throw the information away, and it explains itself only once
(`sync.Once`) so a busy service does not repeat the line - while the report
itself is logged every time.

New config key `slack.enabled` (`SLACK_ENABLED`), false in
`env-example/app.yaml` and in `debug.yaml`. So `APP_ENV=debug ./bin/api` now
boots with no real Slack token, and the local `init.go` edit in
`~/work/stream/aioz-stream` is no longer needed once that checkout catches up.

Test gotcha: asserting on log output, `strings.Count(out, "content report")`
also matches the phrase inside the explanation line. Count the exact
`msg="content report"` instead.


**Update 2026-09-10 (`~/work/stream/aioz-stream`, branch `feat/migrate-with-m3u8` == `origin/stag`).**
That checkout has caught up with the refactor: it now reads **YAML**, not env.
`cmd/http/init.go:199` loads `./${APP_ENV}.yaml`, so the `debug.env` sitting in the
repo root there is dead weight. `internal/config/config.go` is byte-identical to the
treehouse worktree's, so its `debug.yaml` was copied over verbatim and verified by
`TestDebugYamlLoads` (that test skips when the file is absent, so it only really runs
once a developer has one). The `.gitignore` is a `*` catch-all with `!` re-includes, so
`/debug.yaml` is ignored without needing the explicit rule the refactor added.

**`make migrate-up` fails on a pre-migration dev database, and forcing is the fix.**
This DB was built by the old `AutoMigrate`, so migration 1 dies with
`multiple primary keys for table "actions"` and migration 2 with
`relation "idx_users_email_active" already exists`. Only two migrations exist
(`000001_baseline`, `000002_triggers`) and the DB already reflects both, so
`./bin/migrate force 2` marks it correctly with no drift - the baseline's own header
documents this. Result: `version=2 dirty=false`.

**New boot blocker, same shape as the old Slack one: the CDN.**
`storage.MustNewCdnHelper` (`internal/utils/storage/cdn.go:82`) does an eager
`GET {cdn_url}/getBalance` in the constructor and panics on failure. `cdn_url`/`hub_url`
are placeholders (`http://cdn-url`) in `debug.yaml`, `debug.env` AND
`env-example/app.yaml` - no real value is recorded anywhere in the repo, and no
alternative storage backend is wired into `init.go`. Everything before it boots clean
(Postgres, Redis, RabbitMQ, token issuer, and Slack, which is now genuinely disabled by
`slack.enabled: false`). User chose to supply the real CDN URLs in their own config
rather than have the helper made non-fatal, so **the eager-panic pattern is still
there** for the next person.

**How to apply:** in that checkout, `debug.yaml` is already correct apart from the two
CDN URLs; `bin/api` builds with `go build -tags purego`. Do not regenerate config from
`debug.env` there any more.


**2026-09-10: stream Postgres recreated from scratch (`~/work/stream/aioz-stream`).**
User asked to remove the local stream DB and bring up a new one from the env config, and
chose **no backup**. The old DB was 721 MB (176 media, 3 users), all discarded. Steps:
`docker compose --profile vod stop postgres && rm -f postgres`, empty the root-owned
`./postgres_data` bind mount with a throwaway alpine container, then
`docker compose --profile vod up -d --build postgres`. It initialises from `app.env`
(`admin` / `video-db`, port 5437), which `debug.yaml` already matches.

- **`make migrate-up` needs `APP_ENV=debug` passed explicitly.** `cmd/migrate/main.go`
  defaults to `APP_ENV=app`, but this checkout has no `app.yaml` (only `app.env`), so it
  panics with `can not read config file "./app.yaml"` before connecting.
  `APP_ENV=debug make migrate-up` gets to `version=2 dirty=false`.
- **"Drift" explained (checked same day): nothing on stag is lost.** The old DB's 70 tables
  were 56 baseline + `schema_migrations` + 5 tables that the `aioz-payment` dependency
  AutoMigrates at boot (`payment_marks`, `tx_ins`, `tx_outs`, `wallets`, `transactions`: outside
  `internal/migrations`, recreated whenever `bin/api` starts) + 8 tables that only exist on other
  branches: `highlight_segments` (feat/highlight-upload) and `blockchain_logs`,
  `payment_seller_reports`, `paywall_buyers`, `paywall_invoices`, `paywall_pay_per_view_media`,
  `paywall_plans`, `paywall_subscription_playlists` (origin/develop, feat/paywall). HEAD code
  references none of the 8. How the paywall tables got in after the 2026-07-24 wipe is unproven:
  there's no local checkout of a paywall branch after that date in either worktree. The DB listens
  on 0.0.0.0:5437, and another host (10.0.0.190) runs a stream instance against it, so that is the
  likely writer.


**Update 2026-09-10 (refactor worktree, after [[single-binary-cli]]).** Much of the above is stale for the refactor branch. There is one binary, `bin/aiozstream`, and config is YAML at `./config.yaml` (was `debug.yaml`, and before that `debug.env`). `APP_ENV` is gone, and `--config-dir` selects the directory. Run with `make run` / `make run-grpc`. The Slack blocker is gone (it's disabled and logs instead). The current boot blocker is the placeholder `cdn_url`; see [[single-binary-cli]] for the stub.
