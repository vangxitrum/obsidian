---
type: decision
tags: [docker, compose, debug, monkit, pprof, observability, aioz-stream]
created: 2026-09-11
agent: main
---

The debug server (monkit `/metrics`, `/debug/pprof`, `/health`, `/version/`, `/logging`) is exposed from the compose `api` service. Uncommitted as of 2026-09-11.

**What changed:**
- docker-compose.yml, api service: `DEBUG_ADDR: 0.0.0.0:6060` and port `127.0.0.1:${DEBUG_HOST_PORT:-6060}:6060`.
- api.Dockerfile: `EXPOSE 8080 6060`.
- Only `aiozstream api` starts the debug server (internal/cli/api/api.go). gRPC has none.

**Why it's this shape:**
- The config default `127.0.0.1:6060` is the *container's own* loopback, so a published port cannot reach it. The container has to bind `0.0.0.0`. This was proven with a negative control: the server listened inside the container, and the host got curl 000.
- The port is published on host loopback only, because pprof serves heap dumps and CPU profiles. Remote scraping goes through an SSH tunnel or a firewall rule. `DEBUG_HOST_PORT` exists because the user's main checkout runs an API on host :6060.
- The debug server starts only after full wiring (DB, Redis, RabbitMQ, CDN, and the payment gateway's TLS call to eth-dataseed.aioz.network), so a boot failure means no debug port at all.

**E2E recipe (compose `extends` of the real service), and its traps:**
- `extends` inherits `profiles: [vod]`, which hides the service without `--profile`, and `depends_on: postgres/rabbitmq`, which fails with "no such service". It also inherits `container_name: aioz-stream-api`. Override with `profiles: !reset []`, `depends_on: !reset []`, and your own container_name.
- The plain `ubuntu:22.04` image has no CA bundle, so the payment-gateway TLS fails ("x509: certificate signed by unknown authority") and api exits 1. Mount `/etc/ssl/certs/ca-certificates.crt` read-only.
- Relative volumes in the extended file resolve against the repo, so override `/app/input*` targets. Otherwise docker creates root-owned dirs in the worktree.

Related: [[monkit-observability-migration]], [[single-binary-cli]], [[upstream-golang-migrate]].
