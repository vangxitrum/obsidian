# Contract + file tags with usage-by-tag queries (replaces ticket tags)

## Context

An earlier iteration (branch `feat/ticket-tags-usage`, already implemented) let users tag a
**download ticket** and query egress by tag. Checking it against real S3 semantics showed S3 has no
equivalent: presigned URLs are signed client-side and are never a taggable server resource. S3's
actual model is two distinct resources — **bucket tags** (≤50) and **object tags** (≤10, a separate
feature from object user-metadata / `x-amz-meta-*` headers). This change replaces ticket tags with
the S3-faithful pair: **Contract tags** (bucket-equivalent) and **File tags** (object-equivalent,
kept separate from the existing `files.user_metadata`), and reverts tickets to their original
stateless form. Tags are write-once at creation; usage-by-tag covers **storage byte-hours + egress**
at both levels. Full design: `docs/superpowers/specs/2026-07-14-contract-file-tags-usage-design.md`.

## Step 1 — Revert download tickets to stateless

Tickets became persisted rows only to carry tags; that reason is gone. Undo the ticket-tags work:
- `coord/file/download_ticket.go`: `DownloadTicketContent.TicketID` → back to `FileUUID` (original
  24-byte token = file id + expiry; restore `download_ticket_test.go` accordingly).
- Delete migration `000032_download_tickets`, package `coord/file/ticketcleanup/`,
  `coord/file/download_ticket_store*.go(+test)`, `coord/db/ticket_repo_test.go`,
  `coord/file/ticket_tags_test.go` (validation type moves — see Step 4), and
  `internal/testplanet/ticket_usage_test.go`.
- Remove from `coord/file/store.go`/`coord/db/file_store.go`/`coord/db/file_repo.go`:
  `CreateTicket`, `GetTicket`, `DeleteTicketsPastPurge`. From `coord/file/service.go`:
  `IssueDownloadTicket`, `GetTicket`. From `coord/db/accounting_repo.go`: `ticketEgressRollup`,
  `AddTicketEgress`, `ListTicketEgress`, `SumEgressByTag` (its logic is re-derived per-tag below).
- `coord/file/endpoint.go`: `GetDownloadTicket`/`GetDownloadInfoByTicket` revert to pre-tags shape;
  drop the `ticketID *vo.UUID` param from `buildManifest`/`recordDownloadBandwidth` (replaced by
  file-level egress recording in Step 3); remove `GetDownloadTicketRequest.tags` from the proto.
- `AccountingService`: remove `GetTicketUsage` + its request message; wiring in
  `coord/peer.go` `setupTicketCleanup`, the `File.TicketCleanup` field/construction, and
  `TicketCleanup` in `coord/config.go`; the `TicketStore` dep on `accounting.Endpoint`.

## Step 2 — Tag columns (one new migration)

- `contracts.tags JSONB` + GIN index; `files.tags JSONB` + GIN index (a **new column**, distinct
  from `files.user_metadata`/`system_headers`).
- `coord/storage/contract.go`: `Contract` gains `Tags map[string]string`
  (`gorm:"column:tags;serializer:json"`).
- `coord/file/file.go:25`: `File` gains `Tags map[string]string` alongside `SystemHeaders`/
  `UserMetadata` (same gorm serializer pattern).

## Step 3 — Set tags at creation + record file egress

- Proto: add `map<string,string> tags = N` to `CreateContractRequest`
  (`pkg/pb/coord/storage/v1/storage.proto`) and `CreateFileRequest`
  (`pkg/pb/coord/file/v1/file.proto`); regenerate with `buf generate`.
- Validate + persist: `StorageUsecase.CreateContract` (`coord/storage/service.go:42`) and
  `Service.Create` (`coord/file/service.go:62`) normalize+validate tags (Step 4) and set them on the
  new row before insert.
- New `file_egress_rollups` (`file_id, interval_start, egress_bytes`, PK `(file_id, interval_start)`,
  upsert-add) in the Step 2 migration. New `AddFileEgress` in `coord/db/accounting_repo.go` mirroring
  `AddContractEgress` (`:309`); called from `recordDownloadBandwidth` (`coord/file/endpoint.go:805`)
  right beside the existing `AddContractEgress` call, keyed by the file id already in scope.

## Step 4 — Shared tag validation

Rename `coord/file/ticket_tags.go`'s `TicketTags` → a general `Tags` type in `coord/file/tags.go`
(S3 limits already correct: max 10, key ≤128 B, value ≤256 B, CRLF/control-char guard,
case-preserving). Reuse it for both file tags and contract tags (contract imports it from
`coord/file`, or move the type to a small shared leaf package if an import cycle appears — verify at
implementation).

## Step 5 — Preserve deleted-file storage (finalized ledger)

Files are hard-deleted (`DeleteFile` CTE, `coord/db/file_repo.go:102`), so storage history must be
captured at delete time:
- New `file_storage_finalized` (`file_id, contract_id, tags JSONB, encrypted_size, created_at,
  deleted_at`) in the Step 2 migration.
- Extend the `DeleteFile` CTE to `SUM(encrypted_size)` over the segments being deleted and insert the
  finalized row (file's tags + total size + lifespan) in the same transaction as the hard delete.

## Step 6 — Usage-by-tag query RPCs

Extend `UsageByTagResponse` to carry `{tag_value, storage_byte_hours, egress}` (not egress-only). New
RPCs on `AccountingService` (`pkg/pb/coord/accounting/v1/accounting.proto`), handlers in
`coord/accounting/endpoint.go` reusing `PeerIdentityFromContext` + the `contract.OwnerId.Equal(caller)`
guard (`:60`), backed by `UsageService` (`coord/accounting/usage.go`) + repo queries
(`coord/db/accounting_repo.go`), reusing `integrateByteHours` (`coord/accounting/accounting.go:160`):
- `GetUsageByContractTag(tag_key, tag_value?, from, to)` — **no contract_id**; scoped to every
  contract the caller owns (mirror `ListAccountTallies`/`ListAccountEgress`' `JOIN contracts ON
  owner_id` at `accounting_repo.go:429/480`), joined through `contracts.tags`. Reuses the existing
  `contract_storage_tallies` + `contract_egress_rollups` — no new tracking.
- `GetUsageByFileTag(contract_id, tag_key, tag_value?, from, to)` — ownership-checked on the
  contract. Storage = UNION of live files (`files JOIN segments`, byte-hours over
  `[max(created_at,from), min(now,to)]`) and finalized files (`file_storage_finalized` over
  `[max(created_at,from), min(deleted_at,to)]`), matched by `tags`. Egress = `file_egress_rollups`
  joined through `files.tags`. Empty `tag_value` → grouped per value; set → single filtered sum.

## Out of scope (flagged)

- Tag mutability (write-once at creation only; a `SetContractTags`/`SetFileTags` RPC can follow).
- Preserving a deleted file's *egress* rollups (only storage is finalized; downloads of deleted
  files are a narrower edge case).
- Client plumbing (uplink/go-sdk setting `tags` on create) — coordinator side is fully testable via
  gRPC/testplanet regardless.

## Verification (Postgres-backed, live `coord-db` container on :5445, `DEPIN_TEST_POSTGRES`)

- DB-integration (`coord/db`): contract-tag usage grouping/filtering over seeded existing tallies;
  `AddFileEgress` upsert-add; file storage UNION across live + finalized; `DeleteFile` writes a
  correct `file_storage_finalized` row (tags + summed encrypted_size + lifespan).
- Unit: shared `Tags.Validate` limits (move the existing `ticket_tags_test.go` cases).
- End-to-end (`internal/testplanet`): create a tagged contract + tagged file → real
  upload/download → `GetUsageByContractTag` and `GetUsageByFileTag` return correct storage+egress,
  grouped and filtered → cross-owner query rejected (`PermissionDenied`) → delete the file, then
  re-query file-tag storage and assert the finalized entry still contributes.
- `go build ./...`, `go vet ./coord/...`, `go test ./coord/... ./internal/testplanet/...`. (Symlink
  `../go-sdk` into the worktree first — this treehouse worktree lacks the sibling checkout that
  `internal/testplanet` needs: `ln -s /home/tuan/work/depin-workspace/go-sdk
  /home/tuan/.treehouse/depin-b971d9/1/go-sdk`.)
