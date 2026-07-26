---
type: fact
tags: [depin, contract, file, usage-accounting, s3-tags, coord, import-cycle]
created: 2026-07-14
agent: main
---

Replaces [[ticket-tags-usage]]. Implemented S3-faithful **contract tags**
(bucket-equivalent) + **file tags** (object-equivalent, distinct from
`files.user_metadata`/`x-amz-meta-*`) + tag-based usage queries, on branch
`feat/ticket-tags-usage` (uncommitted — no-auto-commit rule). Download tickets
reverted to their original stateless form (`DownloadTicketContent.TicketID` back
to `FileUUID`). Executed via the `execute-plan-verbatim` skill against a
pre-written plan + design doc
(`docs/superpowers/specs/2026-07-14-contract-file-tags-usage-design.md`).

**Design**: tags set once at creation (`CreateContract`/`CreateFile`), write-once
(no update RPC yet). Usage = storage byte-hours + egress, both queryable
grouped-by-tag-value or filtered-to-one-value.

**Key pieces**:
- Migration (renumbered `000032_contract_file_tags`, since the old
  `000032_download_tickets` was deleted in the same pass): `contracts.tags` /
  `files.tags` JSONB + GIN index; new `file_egress_rollups` (mirrors
  `contract_egress_rollups`, keyed by file_id); new `file_storage_finalized`
  (file_id, contract_id, tags, encrypted_size, created_at, deleted_at) — captures
  a file's storage-billing lifespan at hard-delete time so its contribution
  survives the row being removed. Deliberately has NO FK to `files.id` (the whole
  point is to outlive that row); contract_id DOES FK `contracts` since contracts
  are never deleted in this codebase.
- `coord/file/service.go`'s `DeleteFile` → `coord/db/file_repo.go`'s `DeleteFile`
  CTE extended: sums `encrypted_size` over the segments being deleted and inserts
  the finalized row in the same statement as the hard delete. Relies on Postgres
  executing every data-modifying CTE in a `WITH` clause even when the final
  `SELECT` never references it (same pattern already used by the pre-existing
  `DeleteExpiredPendingFiles` query's `deleted_segments` CTE) — verified this is
  real, not assumed.
- New RPCs on `AccountingService`: `GetUsageByContractTag(tag_key, tag_value?,
  from, to)` — **no `contract_id`**, scoped to every contract the caller owns
  (mirrors `GetClientUsage`'s account-wide branch, no explicit ownership check
  needed since the query is inherently owner-scoped) — and
  `GetUsageByFileTag(contract_id, tag_key, tag_value?, from, to)` — contract_id
  required + ownership-checked. Both return `{tag_value, storage_byte_hours,
  egress}` per tag value.
- File-tag storage is NOT tally-based (files aren't periodically snapshotted,
  only contracts/workers are) — it's a direct continuous-time UNION of live files
  (`files JOIN segments`, byte-hours over `[max(created_at,from),
  min(now,to)]`) and finalized/deleted files (`file_storage_finalized`, byte-hours
  over `[max(created_at,from), min(deleted_at,to)]`). File-tag **egress** is
  live-files-only by design — `file_egress_rollups` has `ON DELETE CASCADE` on
  `files.id`, so a deleted file's egress history is gone with it (storage is the
  only thing `file_storage_finalized` preserves; carrying egress through deletion
  was explicitly flagged out of scope).

**Import-cycle lesson (reusable pattern)**: the plan's primary instruction was to
put the shared `Tags` validation type in `coord/file/tags.go` and have
`coord/storage` import it from there. That's impossible: `coord/file` already
imports `coord/storage` (for `ContractStore`), so `coord/storage` importing
`coord/file` back would cycle. The plan *itself* anticipated this exact case
("move to a shared leaf package if an import cycle appears — verify at
implementation") — verified empirically (added the import, watched
`go build` report `import cycle not allowed`) rather than just reasoning it out,
then moved `Tags` to a brand-new leaf package `coord/tags` that both `coord/file`
and `coord/storage` import without conflict. General lesson: when two packages
already have a directed dependency edge and both need a third shared type, that
type needs its own leaf package — don't fight it by trying both import
directions.

**Test-writing gotchas** (bugs in my own test setup, not the implementation —
all caught + fixed during verification):
- `AddFileEgress`/`AddContractEgress` truncate their rollup's `interval_start` to
  the **start of the hour**, which can be up to 59 minutes before the real event
  time. A test window with only a 1-minute margin around a known event timestamp
  can miss the row entirely — need at least an hour of slack.
- The `internal/testplanet` harness seeds *two* placements: the small
  `"testplanet"`-named one (matches whatever `Redundancy:` the test config asks
  for) and the real production RS(29,52,60,80) one (needs 80 workers). Picking
  `ListPlacements()[0]` is not reliable — filter by `Name == "testplanet"`.
- Don't assume egress survives deletion (see above) — a first draft of the e2e
  test wrongly asserted it did.

**Verification**: `coord/db/tag_usage_test.go` (DB-integration, live `coord-db`
Postgres container on :5445, `DEPIN_TEST_POSTGRES`) covers contract-tag
grouping/filtering over seeded tallies, `AddFileEgress` upsert-add, the
live+finalized file-storage UNION, and `DeleteFile`'s finalized-row write.
`internal/testplanet/tag_usage_test.go` is the full e2e: tagged contract + tagged
file, real inline upload/download, two deterministic tally snapshots (via
`tally.Service.SetNow`, not wall-clock, so contract-tag storage byte-hours
integrate to an exact value), grouped/filtered queries on both RPCs, cross-owner
`PermissionDenied` on `GetUsageByFileTag` (not on `GetUsageByContractTag`, which
has nothing to authorize against — a stranger just sees their own, empty,
contracts), then delete-the-file-and-still-contributes. Whole repo
`go build`/`go vet ./coord/...`/`go test ./coord/... ./internal/testplanet/...`
green (Postgres-backed), including one pre-existing unrelated intermittent flake
(`TestSettlementRejectsOverAllocation`, order-settlement test, nothing to do with
this branch — confirmed passing on reruns in isolation and in the full suite).

**Environment gotcha (reused from [[ticket-tags-usage]])**: this treehouse
worktree needs `../go-sdk` symlinked in —
`ln -s /home/tuan/work/depin-workspace/go-sdk /home/tuan/.treehouse/depin-b971d9/1/go-sdk`
— already done, confirmed present before this session started.

**Separately discovered this session**: [[coord-metrics-collection-shipped]]'s
sibling note claiming a "relay observability" feature (relay RTT/rejection/
lifetime/rate metrics, session 9f8fab35) shipped & e2e-verified turned out to be
**absent from this treehouse worktree** — no matching files, no branch, no commit
anywhere in this repo's history. Likely explanation: uncommitted work in a
different, now-torn-down worktree. General lesson: a memory note saying "shipped,
not committed" is a claim about a point-in-time working tree that may no longer
exist — verify the actual files before treating "shipped" as still true,
especially across different treehouse worktree instances of the same repo.
