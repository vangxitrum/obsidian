---
type: plan
project: aioz-stream
created: 2026-09-10
status: approved
---

# Use upstream golang-migrate, drop `aiozstream migrate` and gorm AutoMigrate

## Context

Schema today: embedded SQL in `internal/migrations` applied by our own wrapper (`aiozstream migrate up|down|version|check|force`, `internal/cli/migrate`, `internal/utils/migrate`), run in CI `migrate-job` by scp-ing the app binary to the host. On top, gorm AutoMigrate is still live:

- Every repo constructor in `pkg/v1/repositories/` takes `init bool` and runs `db.AutoMigrate(...)` (38 constructors, bodies AutoMigrate-only).
- **Bug:** `internal/cli/grpcserver/wire.go:133` sets `init := true`, so each gRPC boot AutoMigrates media, media_files, media_thumbnails, media_captions, cdn_files, live_stream_medias, outside `schema_migrations`. gorm infers `bytea` for aioz-common uuid vs schema `text`, so it can try to ALTER id types.
- `repositories.AutoMigrateAll` feeds only `internal/tools/baselinegen` (one-shot, 000001 frozen) and the drift test.

Decisions (user): use the upstream golang-migrate CLI instead of our cmd; DB URL from a CI/CD variable; CLI runs in the CI job directly; no pre-check (drop `migrate check`); also remove all AutoMigrate.

## Steps

0. **Reproduce the gRPC bug end-to-end first.** Throwaway `postgres:16`, apply migrations, `pg_dump -s` before; boot `aiozstream grpc` (rabbitmq container + CDN stub per memory note single-binary-cli); `pg_dump -s` after; record diff or boot panic.

### A. Remove our migrate command

1. Delete `internal/cli/migrate/` and `internal/utils/migrate/`.
2. `cmd/aiozstream/main.go`: drop import + `migrate.Command(g)`, update the package doc usage line (line 8).
3. `cmd/aiozstream/main_test.go`: `TestCommandTree` expects `api, grpc, version` and drops the migrate-subcommand block; `TestConfigDirIsGlobal` paths become `{"api"}, {"grpc"}`.
4. Keep `internal/migrations` (`FS`, `Dir`, SQL, `migrations_test.go`) - tests still embed it; upstream CLI reads the same dir via `-path`.
5. Integration tests (`internal/migrations/apply_integration_test.go`): replace `migrate.New(...)` from utils with a small test helper using the library directly: `iofs.New(migrations.FS, migrations.Dir)` + `database/postgres.WithInstance` over `sql.Open("pgx", dsn)` + `migrate.NewWithInstance`; `Up()` tolerating `ErrNoChange`, `Version()` tolerating `ErrNilVersion`. Any test that exercised `Check()` is deleted.
6. `go mod tidy` (golang-migrate stays, used by tests).

### B. CI `migrate-job` (`.gitlab-ci.yml` ~285-360)

- `image: migrate/migrate:v4.19.1` with `entrypoint: [""]`; no `needs: build-job` artifacts, no `.ssh-target`, no scp/ssh. Job uses the repo checkout.
- Rules add `DATABASE_URL: $PROD_DATABASE_URL` / `$STAG_DATABASE_URL` next to `DEPLOY_PATH`. `before_script` fails early if `DATABASE_URL` is unset.
- Script:
  ```sh
  m() { migrate -path internal/migrations -database "$DATABASE_URL" "$@"; }
  v="$(m version 2>&1 || true)"; echo "before: $v"   # upstream prints version on stderr
  case "$v" in *dirty*) echo "ERROR: schema dirty; fix by hand then: migrate -path internal/migrations -database \$DATABASE_URL force <version>"; exit 1;; esac
  m up
  m version 2>&1
  ```
- Rewrite the job/header comments: remove `migrate check` and config.yaml/compose-dir text; keep the N-1 compatibility note. Update the build-job comment at ~line 231 (no longer "reachable as `aiozstream migrate up`").
- **Prerequisite (ops):** create masked+protected `PROD_DATABASE_URL`, `STAG_DATABASE_URL` (`postgres://user:pass@host:port/db?sslmode=...`), and the GitLab runner must reach both Postgres hosts.

### C. Makefile + docs

- Replace `migrate-up|down|version|check` targets with upstream CLI, pinned to go.mod's version, no install needed:
  `MIGRATE = go run -tags postgres github.com/golang-migrate/migrate/v4/cmd/migrate@v4.19.1 -path internal/migrations -database "$(DATABASE_URL)"`
  targets `migrate-up`, `migrate-down` (`down 1`), `migrate-version`, `migrate-force` (`force $(VERSION)`), `migrate-create` (`create -ext sql -dir internal/migrations -seq -digits 6 $(NAME)`, matches `namePattern` in `migrations_test.go`). Guard: fail if `DATABASE_URL` unset. Drop `migrate-check`, fix `.PHONY`, fix the `build` comment (line 99-100).
- `config.yaml:8`, `internal/cli/api/wire.go:244` comment, `000001_baseline.up.sql` header (lines 5, 11, 16): say `migrate -path internal/migrations -database ... up` / `make migrate-up`, not `aiozstream migrate up`.

### D. Remove gorm AutoMigrate

1. Every constructor in `pkg/v1/repositories/*.go` (`MustNew*Repository`, `New*Repository`, `MustWatermarkRepository`, `exclusive_code.go`): drop `init bool` and the AutoMigrate block.
2. Call sites: `internal/cli/api/wire.go` (drop `const init = false`), `internal/cli/grpcserver/wire.go` (drop `init := true`), `pkg/v1/repositories/factory.go`, `internal/seeds/seed.go`, `internal/app/highlight/{integration,rabbitmq_integration}_test.go`.
3. Replace `pkg/v1/repositories/automigrate.go` with `models.go`: `func Models() []any` listing every model the constructors AutoMigrated (ApiKey, AiGenerationTask, CdnFile, CdnUsageStatistic, EditorProject, EmailConnection, ExclusiveCode, JoinExclusiveProgramRequest, MediaFormat, HighlightChunk/Clip/Manifest/Media, IpInfo, LiveStreamKey, Multicast, Statistic, LiveStreamMedia, Mail, MediaImportTask, MediaSummary, Part, PaymentLog, PlayerTheme, Playlist/Item/Thumbnail, MediaQuality/File, ContentReport, Session/SessionMedia/Action/WatchInfo, MediaStream, Thumbnail/Resolution/File, Usage/UsageLog, User/SubscribeInfo, Media/MediaFile/MediaThumbnail, MediaCaption/File, MediaChapter/File, MediaUsage, WalletConnection, Watermark/MediaWatermark/WaterMarkFile, Webhook/WebhookRetry).
4. `TestModelsMatchTheMigratedSchema`: one scratch DB migrated via the new helper; model columns from `schema.Parse(m, &sync.Map{}, schema.NamingStrategy{})` - `s.Table` + `s.DBNames` where `!FieldsByDBName[n].IgnoreMigration` (embedded prefixes flattened, `gorm:"-"` skipped, no many2many in domain). Same two-way compare, no types. Drop gorm driver/logger imports.
5. Delete `internal/tools/baselinegen/`.
6. Guardrail `internal/tools/automigrate_test.go`, same walk as `embed_test.go` (reuse `repoRoot`, `rel`, `isVendored`, `moduleCacheDir`): fail on `.AutoMigrate(` in any `.go` file.

Out of scope: payment lib creates its own tables at boot (external).

## Verification

- `go build ./...`, `go vet ./...`, `go vet -tags integration ./...`, `make lint`, `go test ./...`.
- `MIGRATE_TEST_DSN=<throwaway pg> go test -tags integration ./internal/migrations/...` passes.
- Mutation checks: remove `Multicast.TableName()` -> drift test fails; add stray `db.AutoMigrate(` -> guardrail fails; revert both.
- Upstream CLI e2e on a throwaway DB: `DATABASE_URL=... make migrate-up` -> version 3; `make migrate-version`; `make migrate-down` -> 2; `make migrate-up` again; `make migrate-create NAME=x` produces `000004_x.{up,down}.sql` then delete them.
- CI script dry-run locally: `docker run --rm --network host -v $PWD:/w -w /w --entrypoint sh migrate/migrate:v4.19.1 -c '<job script>'` with `DATABASE_URL` set, incl. a forced-dirty DB to confirm the guard exits 1. `gitlab-ci` YAML lint via `glab ci lint` if available.
- Step 0 rerun: `pg_dump -s` before/after `aiozstream grpc` boot -> empty diff; `aiozstream api` boots too.
- `./bin/aiozstream migrate` -> unknown command.

## After

- No commit. Update memory note `.agent/memory/aioz-stream/legacy-tables-drop-candidates.md` + `single-binary-cli.md` (migrate cmd gone, upstream CLI, DATABASE_URL vars, grpc AutoMigrate bug). Hermes writes this plan to `Projects/aioz-stream/plans/2026-09-10-upstream-golang-migrate.md` + INDEX link.
