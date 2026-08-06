---
type: decision
tags: [depin, testing, testplanet, delete, qa, migrations, grpc]
created: 2026-08-05
agent: main
---

Implemented section 18 "Deletion (DEL)" of [[testplanet-coverage-plan]] on branch
`feat/implement-testcases` (worktree `/home/tuan/.treehouse/depin-b971d9/4/depin`).
**Uncommitted** - global CLAUDE.md rule, never commit unless asked.

Delivered: `internal/testplanet/delete_file_test.go` (9 tests), 5 new tests in
`delete_contract_test.go` + its fixture upgraded from an inline segment to a real
RS upload, 19 `TC-DEL-*` rows in `docs/test-plan.md`, 4 `Uplink` wrappers
(`CreateContract`/`UploadToContract`/`DeleteFile`/`DeleteContract`) and
`Coord.RepairQueue()` + `Coord.RepairOnce()` in `helpers.go`.
**All 15 pass** (`ok aioz-depin/internal/testplanet 62.475s`).

## Two pre-existing bugs the work exposed

**1. Duplicate migration `000033` on `origin/develop` - broke EVERY testplanet test.**
`000033_worker_tags` (2026-07-24, worker-tags merge) and `000033_worker_version`
(2026-07-31, comopent-tags merge) both took version 33; golang-migrate refuses the
set outright:
`migration error: failed to init driver with path migrations: duplicate migration file: 000033_worker_version.up.sql`.
Not a test problem - no planet could boot. User chose the simple renumber:
`worker_version` -> `000040`. **Open follow-up:** any DB whose lineage applied
`worker_version` as 33 will now skip `000033_worker_tags` forever. The local
`coord-db` (port 5445) is exactly that case - schema_migrations version 38, has the
`workers.version` column, has NO `worker_tags` table and no `is_trusted` column.
That migration needs applying by hand there. The alternative considered and
rejected was keeping 33 and adding a converging `000040` holding BOTH DDL blocks
(all statements are `IF NOT EXISTS`, so that converges every lineage).

**2. `vo.GrpcError` had no `Forbidden` case** (`internal/vo/errors.go`), so every
authorization failure routed through it returned `Internal`, not `PermissionDenied`.
Caught by `TestDeleteFileRejectsNonOwner`:
`code = Internal desc = file: forbidden`. Clearly an omission, not a design choice -
its HTTP twin `GetHttpStatusCodeFromError` maps `Forbidden` -> 403, and
`coord/storage/endpoint.go:68` hand-patched the same mapping locally (which is why
the *contract* non-owner test passed while the *file* one failed). Fixed centrally
with one `case`, so all endpoints benefit; no test asserted the old behaviour.
Matches [[../\_global/index|the "fix it in the shared place" preference]] - see also
the standalone `feedback_sdk_owns_safety_defaults` memory.

## Facts pinned by the tests

- `DeleteFile`'s SQL is idempotent but **the RPC is not**: `AuthorizeFileOwner`
  returns NotFound before the store's zero-rows no-op is reached, so a second
  delete errors.
- That same auth path resolves the contract via the **live-only** `GetByID`, so a
  file under a soft-deleted contract is NotFound (and only `DeleteContract` can
  drain it).
- Neither delete path touches worker pieces; asserted via
  `w.Storage.HashStoreBackend.SpaceUsage().UsedForPieces` before and after.
- After a file delete the NEXT tally writes an explicit **zero** row for the
  emptied contract (`fillContractTallies` zero-seed) - that, not the delete, is
  what stops byte-hour billing.
- `repair_queue` has no FK to `segments`; `SegmentRepairer.Repair` treats the
  lookup miss as done and drops the job.
- `deleteBatchSize = 100`; a 101-file contract exercises the second short batch.

## Traps

- **`go build ./...` cannot pass on this branch at all**, unrelated to any of this:
  `cmd/uplink` imports the `uplink/` package that commit `63564ce`
  ("refactor: remove keytool") deleted. Build
  `./internal/... ./coord/... ./worker/...` instead.
- `segment_pieces` and `segment_piece_uploads` **do not exist** - later migrations
  dropped them; pieces live in a `bytea` column on `segments`. Do not write
  assertions against those tables.
- Adding a download to `TestDeleteContract`'s force=false step silently broke the
  later `preUsage.Egress == postUsage.Egress` check - that download is real egress
  recorded after the snapshot. Order matters: rejection step before usage snapshot.
- Pre-existing failures to expect, both confirmed failing with this work stashed:
  `TestEdgeserverMetadataHeaders` (asserts an empty `Cache-Control` the handler
  always sets) and `internal/vo`'s `TestPieceIDScanNullAndEmpty`.
- `docs/test-plan.md` still contains three rows naming code the repo does not have:
  TC-FS-11's `task.go` chore, TC-USG-06's `file_storage_finalized` table, and
  TC-FS-03's multipart upload. Flagged in the plan, rows not yet rewritten.

Related: [[testplanet-coverage-plan]], [[delete-contract-shipped]],
[[delete-file-phase1]], [[edgeserver-metadata-headers]], [[depin-test-plan]]
