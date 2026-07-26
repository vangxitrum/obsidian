---
title: Contract + file tags with usage-by-tag queries
date: 2026-07-14
project: depin
status: contract tags shipped; file tags deferred
branch: feat/ticket-tags-usage
---

# Contract + file tags with usage-by-tag queries (replaces ticket tags)

> **Implementation status (2026-07-16):** shipped as **contract tags only**.
> File (object-level) tags — the `files.tags` column, `file_egress_rollups`,
> `file_storage_finalized`, and the `GetUsageByFileTag` RPC — are **deferred /
> out of scope for this cut** and are NOT in the source tree. The whole file-tag
> design below is retained as the record for when it is picked back up. What
> actually shipped: `contracts.tags` (bucket-equivalent), set at contract
> creation, plus `GetUsageByContractTag(tag_key, tag_value?, from, to)` scoped to
> every contract the caller owns. Verified via DB-integration + a real
> `internal/testplanet` e2e (`TestContractTagUsage`).

## Context

The previously-shipped feature let users tag a *download ticket* and query egress
by tag. Comparing against real S3 semantics surfaced that S3 has no equivalent at
all: presigned URLs are signed client-side and never get a server-minted taggable
identity. S3's actual tagging model is two distinct, well-known resources:

- **Bucket tags** (up to 50) — the account/billing-grouping level.
- **Object tags** (up to 10, via `PutObjectTagging` — a *different* feature from
  object user-metadata/`x-amz-meta-*` headers, which this codebase already has as
  `files.user_metadata`).

This design replaces ticket tags with the S3-faithful pair: **Contract tags**
(bucket-equivalent) and **File tags** (object-equivalent, kept separate from the
existing `user_metadata` field). Tickets revert to their original stateless design.

## Revert: download tickets back to stateless

The only reason tickets became persisted rows was to carry tags. With tags moving
to contract/file, that statefulness is no longer needed:

- `DownloadTicketContent.TicketID` reverts to `FileUUID` (original 24-byte token:
  file id + expiry, no DB row).
- Drop `download_tickets` / `ticket_egress_rollups` tables (migration
  `000032_download_tickets`), the `coord/file/ticketcleanup` chore, its
  `coord/config.go`/`coord/peer.go` wiring, `Service.IssueDownloadTicket` /
  `Service.GetTicket`, and the `AccountingService.GetTicketUsage` RPC.
- `GetDownloadTicketRequest.tags` field removed; `GetDownloadTicket` /
  `GetDownloadInfoByTicket` return to their pre-ticket-tags shape
  (`coord/file/endpoint.go`, `coord/file/service.go`).
- Egress recording (`recordDownloadBandwidth`) drops the `ticketID *vo.UUID`
  param entirely — replaced by file-level egress recording (see below).
- Delete `coord/db/ticket_repo_test.go`, `coord/file/ticket_tags_test.go`,
  `coord/file/download_ticket_store*.go(+test)`, `coord/file/ticketcleanup/`,
  `internal/testplanet/ticket_usage_test.go`.

## Data model

### Contract tags (bucket-equivalent) — SHIPPED

- `contracts.tags JSONB` + GIN index (new migration).
- `coord/storage/contract.go`: `Contract` gains `Tags map[string]string` (Go-only
  field, `gorm:"column:tags;serializer:json"` — same pattern as `File`'s
  `SystemHeaders`/`UserMetadata`).
- Set once at creation: `CreateContractRequest.tags` (proto,
  `pkg/pb/coord/storage/v1/storage.proto`) → validated → `StorageUsecase.CreateContract`
  (`coord/storage/service.go`).
- Contracts are never deleted anywhere in this codebase today (no delete path
  exists for `contracts`), so contract-tag usage has no retention/GC concern —
  it's always queryable from the contract's existing tallies.
- Validation lives in a shared leaf package `coord/tags` (the `Tags` type), NOT
  under `coord/file` as originally planned — `coord/file` already imports
  `coord/storage`, so a shared type in either would import-cycle; `coord/tags` is
  imported by both without conflict.

### File tags (object-equivalent) — DEFERRED (not implemented)

- `files.tags JSONB` + GIN index (same migration) — a **new, separate** column
  from `files.user_metadata`/`system_headers`, matching S3's real distinction
  between Object Tagging and custom metadata headers.
- `coord/file/file.go`: `File` gains `Tags map[string]string` alongside the
  existing `SystemHeaders`/`UserMetadata` fields.
- Set once at creation: `CreateFileRequest.tags` (proto,
  `pkg/pb/coord/file/v1/file.proto`) → validated (reuse the shared `coord/tags`
  S3 limits: max 10, key ≤128B, value ≤256B, CRLF guard, case-preserving) →
  `Service.Create`.

## Usage tracking

### Contract-tag usage — reuses existing tallies, no new tracking — SHIPPED

`contract_storage_tallies` and `contract_egress_rollups` already exist and are
written independently of tags. A new query joins them through `contracts.tags`:

- `GetUsageByContractTag(tag_key, tag_value?, from, to)` — **no `contract_id`
  param**; caller is implicit (mTLS identity), scoped to every contract *they
  own* (mirrors `GetAccountUsage`'s account-wide pattern). Empty `tag_value` →
  grouped breakdown per value; set → single filtered sum. Returns
  `{storage_byte_hours, egress}` per tag value (extends `UsageByTagResponse` to
  carry both, not just egress). Two contracts with the same tag value merge into
  one row (S3 Cost Explorer semantics).

### File-tag usage — new egress rollup + new durable storage ledger — DEFERRED

Egress: mirrors the (now-reverted) ticket egress rollup, but keyed by `file_id`
directly (no ticket indirection needed — `file_id` is already known at every
download site).

- New `file_egress_rollups` table (`file_id, interval_start, egress_bytes`,
  upsert-add), recorded in `recordDownloadBandwidth` alongside the existing
  contract egress call. No FK-cascade/GC concern: rows persist as long as the
  file exists, same lifetime as the file's own tags.

Storage: files are **hard-deleted** today (`Service.Delete` → `DeleteFile` CTE,
`coord/db/file_repo.go`), so a query-time-only calculation
(`encrypted_size × (now - created_at)`) would lose a deleted file's history. Per
your call, storage usage must survive deletion, so:

- New `file_storage_finalized` table: `file_id, contract_id, tags (JSONB copy),
  encrypted_size, created_at, deleted_at`. Written once, at the moment
  `DeleteFile` runs — extend its CTE to also `SUM(encrypted_size)` over the
  segments being deleted and insert the finalized row before/in the same
  transaction as the hard delete (so the file's tags + total size survive its
  own row being removed). Relies on Postgres executing every data-modifying CTE
  in a `WITH` clause even when unreferenced by the final SELECT.
- `GetUsageByFileTag(contract_id, tag_key, tag_value?, from, to)` computes
  storage-hours as a **UNION** of:
  1. **Live files**: `files JOIN segments` where `files.tags` matches, byte-hours
     integrated over `[max(created_at, from), min(now, to)]`.
  2. **Finalized (deleted) files**: `file_storage_finalized` where `tags`
     matches, byte-hours integrated over `[max(created_at, from),
     min(deleted_at, to)]`.
  Egress is summed from `file_egress_rollups` joined through `files.tags` (live)
  — a deleted file's *egress* rollup rows have no surviving tag join once the
  file row is gone; carrying egress rollups through deletion is out of scope for
  this round (flagged below) since egress after a file is deleted is a narrower
  edge case than storage (a file typically isn't downloaded after being deleted).
- `GetUsageByFileTag` takes an explicit `contract_id` (ownership-checked, mirrors
  the original per-contract `GetUsageByTag`) — file-tag queries are scoped
  within one contract/bucket, matching how S3 tagging analysis is normally
  scoped to one bucket at a time. (If cross-contract object-tag aggregation is
  wanted later — like S3 Storage Lens Groups — drop `contract_id` and join
  through `owner_id`, same shape as the contract-tag query.)

## Out of scope (flagged, not blocking)

- **File (object) tags entirely** — deferred for this cut; ship contract tags
  first (see status banner). The file-tag design above is the record for picking
  it back up.
- Tag mutability (update after creation) — write-once at creation only, per your
  call. A `SetContractTags`/`SetFileTags` RPC can follow later if needed.
- Preserving a deleted file's *egress* rollups the same way storage is preserved
  (the finalized table only captures storage). Only worth adding if downloads
  of already-deleted files turn out to matter.
- Client plumbing (uplink/go-sdk setting `tags` on create calls) — same
  follow-up-track reasoning as before; the coordinator side is fully testable
  via gRPC/testplanet regardless.

## Verification

Same rigor as before: DB-integration tests against the live Postgres test
container for every new query (contract-tag grouping/filtering reusing existing
tallies, file egress rollup upsert, file storage UNION across live+finalized,
`DeleteFile`'s finalized-row write), pure-Go unit tests for the shared tag
validation type, and a real end-to-end `internal/testplanet` test exercising
tagged contract/file creation → real upload/download → usage-by-tag RPCs →
ownership rejection → a file delete followed by a storage-by-tag query proving
the finalized entry preserved its contribution.

For the shipped contract-tags-only cut, verification is: `coord/db` DB-integration
`TestAccountingRepositoryContractTagUsage` + the `internal/testplanet`
`TestContractTagUsage` e2e (tagged contract, real upload/download, deterministic
tally snapshots via `SetNow`, grouped + filtered queries, stranger-sees-empty),
plus `go build`/`go vet`/`go test` green across `coord/...` and testplanet.
