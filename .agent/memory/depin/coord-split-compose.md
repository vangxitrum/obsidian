---
type: fact
tags: [coord, docker, deployment, compose, migrate, identity]
created: 2026-07-07
agent: main
---

`docker-compose.coord-split.yml` = production multi-container coord: one container per role
(`api`, `core`, `ranged-loop`, `audit`, `repair`, `relay`) + `postgres`, same image
(`Dockerfile.coord`, prebuilt `bin/coord`+`bin/keytool`). api/audit/repair scale horizontally
(`--scale`), core/ranged-loop/relay are singletons.

**DSN env gotcha (bug fixed 2026-07-07):** coord DB DSN flag is `--app-config.db.postgres-dsn`
(DB nests under `app-config`, `coord/config/config.go:9`). viper env = `AIOZ_APP_CONFIG_DB_POSTGRES_DSN`
(prefix AIOZ, `.`/`-`→`_`, AutomaticEnv). Compose files used wrong `AIOZ_DB_POSTGRES_DSN` → no-op →
fell back to config default `localhost:5445`. Corrected in both docker-compose.coord.yml and
docker-compose.coord-split.yml. Env beats config file (viper precedence).

Note: user ended up REMOVING the init-identity + migrate one-shots from the split file (chose manual
out-of-band bootstrap). x-role-deps trimmed to postgres-only. The `coord migrate` subcommand
(cmd/coord/main.go) still exists in the binary. Below describes the fuller hardened variant:
- New one-shot **`coord migrate`** subcommand (`cmd/coord/main.go`, `cmdMigrate`): ConnectDB +
  `dbConn.Migrate()` (embedded migrations, `coord/db/embed.go`) then exit. `ErrNoChange`=success.
- Two one-shot bootstrap services: **`init-identity`** (`coord setup` + `keytool create` for coord id
  at `/identity` and relay id at `/identity/relay`, idempotent `[ -f ]` guards, `COORD_ID_DIFFICULTY` knob)
  and **`migrate`**. All 6 roles gate on `postgres healthy + init-identity + migrate` via `x-role-deps`
  anchor → no migration race (only core/run migrate in-process; migrate step makes it deterministic).
- Identity driven by **config** (`identity.cert-path`/`identity.key-path`), not `--identity-dir` cmd param.
  `coord-identity:/identity` volume shared by coord peers. **Relay is isolated**: it mounts its OWN
  `coord-relay-identity` volume at `/identity` (NOT coord-identity, so it never sees coord's private keys),
  id at standard `/identity/identity.cert` = config default (no env override). init-identity co-mounts
  `coord-relay-identity` at `/relay-identity` just to mint the relay id.

Bring-up: `COORD_ID_DIFFICULTY=8 docker compose -f docker-compose.coord-split.yml up --build`
(needs bin/coord+bin/keytool prebuilt first). core binds no gRPC port (only debug) → not healthcheck-able,
which is why the one-shot migrate is the gate, not core health.
