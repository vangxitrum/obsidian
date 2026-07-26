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
