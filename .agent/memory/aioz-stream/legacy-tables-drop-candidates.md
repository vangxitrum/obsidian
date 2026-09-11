---
type: decision
tags: [migrations, golang-migrate, postgres, schema, cleanup, automigrate]
created: 2026-09-10
agent: main
---

**No table was dropped between main and `refactor/sync-with-template-source`.** A DB built by main HEAD's AutoMigrate has the same columns, types and constraints as the baseline. The baseline only adds: editor_projects, highlight_chunks/clips/manifests/media, media_import_tasks, and idx_media_mtype/idx_media_offset.

The real leftovers are legacy objects. AutoMigrate never drops anything, so every renamed or removed model stays behind in old DBs.

**How the legacy list was found:**
- Replay AutoMigrate over every first-parent commit on main (238) and stag (11) that touched the models, all into one DB, then diff against the baseline.
- The replay program must swallow per-model errors, so the union DB is pessimistic: its types can be states that prod could never boot on.
- Result: 14 legacy tables + 86 columns, plus cdn_usage_logs, live_streams and live_stream_multicast_models, which appear in main's history but the replay couldn't rebuild.
- The only stag-only object is highlight_segments.

**What shipped (uncommitted on the branch, 2026-09-10):**
- 000001's ADD CONSTRAINTs go through `pg_temp.add_constraint`, which makes it idempotent. Before this, `migrate up` on an AutoMigrate DB failed with "multiple primary keys for table actions" and left v1 dirty. CI would have hit that on prod.
- The helper also replaces a primary key whose definition differs. Old models keyed cdn_files on (file_id, size, offset), and without a unique id every FK to cdn_files(id) fails. It drops NOT NULL on the old PK's columns.
- 000002 does DROP INDEX IF EXISTS before creating the two user unique indexes. main's trigger.sql already created them on every boot.
- 000003_deprecate_legacy renames the legacy objects to `_deprecated_*` and drops NOT NULL on them (parts.video_id, cdn_files.file_id and media_thumbnails.id were NOT NULL with no default). The down migration renames them back. This is phase one of the two-phase approach the user chose; a later 000004 drops everything `_deprecated_`.
- ~~`aiozstream migrate check`~~ REMOVED 2026-09-10 with the whole `aiozstream migrate` cmd (user chose upstream golang-migrate CLI, no pre-check). golang-migrate has no dry-run. See [[upstream-golang-migrate]].
- Old DBs join with `migrate up`, never `force 1`, which would skip the 6 new tables. The Makefile `migrate-baseline` target was removed.

**Traps:**
- golang-migrate runs every file of one `up` on a single connection, so `pg_temp` functions persist across files. Use CREATE OR REPLACE.
- `TestOnlyTheBaselineUsesIfNotExists` greps the literal text, comments included. Inside plpgsql, use PERFORM + NOT FOUND instead.
- The live sub-slice split renamed the structs, so GORM queried `multicasts` and `statistics` (42P01). Fixed with TableName() methods.
- `migrate version` printed with cmd.Printf, which goes to stderr, so the CI dirty-check grep could never fire. See [[cobra-print-goes-to-stderr]].
- The 11 dead repo methods that queried legacy columns (cdn_file/thumbnail/live_stream_video, e.g. GetMediaTotalStorage), plus their 5 store-interface lines, were deleted on 2026-09-10.
- Guard against model/schema drift: `repositories.Models()` (pkg/v1/repositories/models.go, replaced AutoMigrateAll 2026-09-10) is the single list of models, used by `TestModelsMatchTheMigratedSchema` (integration tag), which now reads model columns with gorm `schema.Parse` (no AutoMigrate, no second DB). baselinegen deleted (in git history). The test compares table.column sets, not types, because gorm infers bytea for aioz-common uuid. It was verified to catch the multicast TableName bug.
- The 000002 down used to leave delete_player_logo_file and both idx_users_*_active indexes behind. Fixed.
- zsh doesn't word-split `$VAR` in `for t in $VAR`. A generator script silently wrote one bogus table name. Run generators under bash.

**Never drop** payment_marks, transactions, tx_ins, tx_outs or wallets. The payment lib creates them at boot, and they are not in the baseline.

**Still unknown:** the real prod/stag shape. Run `migrate check` against them before the deploy.

Related: [[single-binary-cli]], [[slice-migration-state]], [[live-sub-slices]].

**Static re-verification (2026-09-10, go/ast over all 1302 origin/main commits + 94 stag-only, gorm v1.25.12 NamingStrategy; script in session scratchpad, throwaway):**
- No NamingStrategy/SingularTable/TablePrefix ever set in any gorm.Config, so default plural snake names apply everywhere. No many2many join tables in history.
- 000003 MISSES 9 real first-parent columns: live_stream_media.{width,height,size,duration,frame_rate,audio_bitrate,hls_url,i_frame,player_url} (LiveStreamMedia at cf97712c 2025-05-23; Assets embedded without prefix).
- 000003's media_thumbnails.{id,file_id,offset,size,resolution,created_at} come from MustNewMediaThumbnailRepository, which had NO caller in its 11 commits (bd633133..4c2a44cb). They are probably absent in prod, but harmless because deprecate_column is IF-EXISTS.
- Side-branch-only (never on main first-parent): cdn_usage_statistics 11 credit/date cols, content_reports.reporter_id, media.job_status, watermarks.water_mark_id, tables cdn_usage_logs + live_streams (live_streams ctor had no caller either).
- live_stream_multicast_models was NEVER AutoMigrated (only used for queries in 09818b0a/b1569cf8); AutoMigrate stayed on LiveStreamMulticast.
- stag: multicasts/statistics structs (699f2cac+) were never AutoMigrated. cmd/http has had `const init = false` since 548c3cbd (2026-09-08), and cmd/grpc (init := true) doesn't build those repos. highlight_segments was AutoMigrated via cmd/http from 6ffdb467 to d89c2d35 (2026-09-05).
- FKs from baseline tables into legacy tables: parts.video_id and playlist_items.video_id -> videos (fk_videos_parts, fk_videos_playlist_items / fk_playlist_items_video). live_stream_statistics.live_stream_key and live_stream_multicasts.live_stream_key -> live_stream_keys (CASCADE). Also formats/streams.media_id -> media, so deleting a media row is blocked while legacy rows reference it.

## Implementation state (2026-09-10 evening)
The real phase-1 migration is `internal/migrations/000003_deprecate_legacy.{up,down}.sql`. It is **uncommitted** in the treehouse worktree `~/.treehouse/aioz-stream-370b08/2/aioz-stream`, branch `refactor/sync-with-template-source`, from another session.

It uses `pg_temp` helper functions:
- **up:** rename to `_deprecated_*` (guarded by to_regclass or information_schema) and DROP NOT NULL
- **down:** rename back, leaving NOT NULL unrestored

It is covered by `TestMigrationsAdoptADatabaseAutoMigrateBuilt` in `apply_integration_test.go`. Do not start a second 000003 on stag: the version would collide.

**Gaps found by the go/ast history replay:**
- Missing `live_stream_media` stale columns, which cf97712c migrated on first-parent: width, height, size, duration, frame_rate, audio_bitrate, hls_url, i_frame, player_url.
- The 6 `media_thumbnails` columns it includes were probably never created, because their constructor was never called. They're harmless under the guards.
- 14 side-branch-only columns are omitted (cdn_usage_statistics x11, content_reports.reporter_id, media.job_status, watermarks.water_mark_id). Include them only if a branch build ever hit prod.

**Phase 2 (drop):** `parts.video_id` and `playlist_items.video_id` may carry foreign keys to `videos`. Drop those columns before `_deprecated_videos`, or use CASCADE.

The history script is `scratchpad/hist/main.go` in session e326c766.
