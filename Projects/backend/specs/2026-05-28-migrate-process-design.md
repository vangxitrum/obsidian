---
type: spec
project: backend
tags: [backend, migration, process, design]
created: 2026-05-28
status: approved
---

# Migration Process Design

**Date:** 2026-05-28  
**Branch:** feat/update-migrate-process  
**Status:** Approved

## Problem

The current schema management has two interleaved systems running at every app startup:

1. **GORM `AutoMigrate`** — `internal/database/migrate.go:Migrate()` — diffs ~60 model structs against the live DB and issues ALTER/CREATE statements. Uncontrolled, no version tracking, runs on every boot.
2. **Raw SQL re-runner** — `MigrateManualUp()` — reads all `.up.sql` files from `internal/database/migration/` via an embedded FS and re-executes them on every startup. No version tracking; SQL must be idempotent. Files have duplicate sequence numbers (two `000006_` and two `000007_` files).

Neither system is suitable for controlled CI/CD deployments.

## Goals

- Remove all schema mutations from app startup.
- Replace with an explicit, versioned migration step in the CI/CD pipeline.
- Track applied migrations so only pending files are executed.
- Keep tooling simple — use the existing `golang-migrate` CLI that is already used for file creation.

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Run mode | Explicit CI/CD step, never at startup | Safer; no uncontrolled schema changes in production |
| Tool | External `golang-migrate` CLI binary | Already used in Makefile; no new Go code needed |
| File format | Flat directory, timestamp-based version IDs | Standard golang-migrate format; avoids duplicate numbering |
| Tracking | `schema_migrations` table (managed by golang-migrate) | Battle-tested; zero custom code |
| Baseline | `pg_dump --schema-only` from live dev DB | Single source of truth; captures GORM + raw SQL state |

## File Structure

Migration files live in `internal/database/migration/` as a flat directory:

```
internal/database/migration/
  20260528000000_baseline.up.sql       ← full schema dump (generated once)
  20260528000000_baseline.down.sql     ← DROP TABLE for all tables
  {timestamp}_{description}.up.sql    ← future migrations
  {timestamp}_{description}.down.sql
```

Naming format: `{YYYYMMDDHHmmss}_{description}.{up|down}.sql`

All old numbered files (`000001_*` through `000008_*`) are deleted as part of migration to this system.

## Makefile Targets

`.env` is sourced at the top of the Makefile via `include .env` + `export`. A `DATABASE_URL` variable in URL format must be present:

```
DATABASE_URL=postgres://postgres:postgres@localhost:5438/aiozmap?sslmode=disable
```

Makefile targets:

```makefile
include .env
export

MIGRATION_DIR = internal/database/migration
MIGRATE      ?= migrate   # external golang-migrate CLI

migrate-up:
    $(MIGRATE) -path $(MIGRATION_DIR) -database "$(DATABASE_URL)" up

migrate-down:
    $(MIGRATE) -path $(MIGRATION_DIR) -database "$(DATABASE_URL)" down $(or $(steps),1)

migrate-version:
    $(MIGRATE) -path $(MIGRATION_DIR) -database "$(DATABASE_URL)" version

migrate-create:
    @TS=$$(date +%Y%m%d%H%M%S) && \
    touch $(MIGRATION_DIR)/$${TS}_$(name).up.sql && \
    touch $(MIGRATION_DIR)/$${TS}_$(name).down.sql && \
    echo "Created $${TS}_$(name).{up,down}.sql"
```

## CI/CD Integration

A `migrate` stage is added to `.gitlab-ci.yml` between `build` and `deploy`:

```yaml
migrate:
  stage: migrate
  script:
    - make migrate-up
  environment:
    name: $CI_ENVIRONMENT_NAME
  variables:
    DATABASE_URL: $PROD_DATABASE_URL   # injected as CI secret variable
```

The production `DATABASE_URL` is stored as a protected CI/CD secret variable, not in `.env`.

## Go Code Changes

### Remove from `cmd/api/main.go` and `cmd/datasource/main.go`

```go
// DELETE these two blocks:
if err := database.MigrateManualUp(db); err != nil { ... }
if err := database.Migrate(db); err != nil { ... }
```

### Delete `internal/database/migrate.go`

The entire file is removed. It contains only `Migrate()` and `MigrateManualUp()`.

### Update `internal/database/db.go`

Remove the embedded FS and `upFS` variable — they are no longer needed at runtime:

```go
// DELETE:
//go:embed migration/*up.sql
var upFS embed.FS
```

## Baseline Generation (One-Time Manual Step)

Run once against the dev database to produce the canonical starting point:

```bash
# 1. Start dev DB
docker-compose up -d db

# 2. Run the app once so AutoMigrate + MigrateManualUp apply the current schema
./bin/api --config config.yaml   # Ctrl+C after startup

# 3. Dump the schema
pg_dump --schema-only \
        --no-owner --no-acl \
        -h localhost -p 5438 -U postgres aiozmap \
        > internal/database/migration/20260528000000_baseline.up.sql

# 4. Write the down file
# Add DROP TABLE IF EXISTS ... CASCADE for all tables,
# DROP TYPE IF EXISTS for custom enums,
# DROP EXTENSION IF EXISTS postgis

# 5. Delete all old migration files
rm internal/database/migration/0000*.sql

# 6. Verify: run migrate-up against a clean DB and confirm app boots
```

## Adding Future Migrations

```bash
# Create a new migration file pair
make migrate-create name=add_user_wallet_column
# creates: 20260601120000_add_user_wallet_column.up.sql
#          20260601120000_add_user_wallet_column.down.sql

# Edit the generated files
# internal/database/migration/{timestamp}_add_user_wallet_column.up.sql
# internal/database/migration/{timestamp}_add_user_wallet_column.down.sql

# Apply locally
make migrate-up
```

## Out of Scope

- GORM is kept for query building — only `AutoMigrate` is removed.
- No changes to model structs or repository layer.
- The geo migration binary (`migrate-geo`) is separate and unchanged.
