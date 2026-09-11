---
type: decision
tags: [migrations, golang-migrate, ci, gorm, automigrate, aioz-stream]
created: 2026-09-10
agent: main
---

On `refactor/sync-with-template-source` (uncommitted, 2026-09-10) the schema is applied only by the **upstream golang-migrate CLI**. The `aiozstream migrate` subcommand, `internal/cli/migrate` and `internal/utils/migrate` are gone, and so is every gorm AutoMigrate call.

**How it runs:**
- Locally: `make migrate-up|migrate-down|migrate-version|migrate-force VERSION=n|migrate-create NAME=x` with `DATABASE_URL=postgres://...`. The Makefile uses `go run -tags postgres .../cmd/migrate@v4.19.1` (pinned to go.mod), so there is nothing to install. It needs Go >= 1.24, and the toolchain auto-switches.
- CI `migrate-job` runs the `migrate/migrate:v4.19.1` image (entrypoint cleared) **in the job itself**, against the checkout's `internal/migrations`, with no ssh or scp. The URL comes from masked CI/CD vars `PROD_DATABASE_URL` / `STAG_DATABASE_URL`, mapped to `DATABASE_URL` by the rules. **Ops prerequisite:** create those vars. The GitLab runner must reach both Postgres hosts.
- There is no pre-check. golang-migrate has no dry-run and does not wrap several files in one tx. The user chose to drop `migrate check`. The dirty-version guard stays.

**Traps:**
- Upstream `migrate version` prints to **stderr** and exits non-zero on a fresh DB (`error: no migration`). The CI script does `v="$(m version 2>&1 || true)"` and then does `case *dirty*`.
- The gRPC service had `init := true` in its wiring, so every boot AutoMigrated 6 tables. Reproduced 2026-09-10: one boot turned `cdn_files.created_by` text -> bytea, because gorm infers bytea for aioz-common uuid. Fixed by removing the `init` param from all 38 repo constructors. Guardrail: `internal/tools/automigrate_test.go` (`TestNothingCallsAutoMigrate`) fails on any `.AutoMigrate(` outside comments.
- The integration tests use the golang-migrate library directly (`newMigrator` helper in apply_integration_test.go), the same version as the CLI.
- E2E recipe used: throwaway `postgres:16` + `rabbitmq:3-alpine` (admin/admin) + python CDN stub. Boot with the env overrides `POSTGRES_*`, `RABBITMQ_*`, `CDN_URL`, `DEBUG_ADDR`, `GRPC_PORT`, then compare `pg_dump -s` before and after.

- 2026-09-11: with AutoMigrate gone the constructors cannot panic, so the `Must` prefix was dropped. The 34 `MustNew*Repository` constructors are now `New*Repository`, and `MustWatermarkRepository` is now `NewWatermarkRepository`. New repo constructors should be `NewXRepository(db *gorm.DB)`.

Related: [[legacy-tables-drop-candidates]], [[single-binary-cli]], [[go-edit-formatter-hook]].
