# Tagged download tickets + egress-by-tag usage queries

## Context

Users want to attach arbitrary key-value tags to a **download ticket** (the presigned-download
bearer token), then query how much **egress** each ticket / each tag consumed. This mirrors
S3 cost-allocation tags: tag a resource, then filter/group usage reports by tag key/value.

Confirmed scope decisions (from user):
- **Entity = download ticket** (not file, not contract).
- **Usage = egress only** (a download ticket only ever drives downloads -> egress; it has no
  storage lifecycle).
- **Tags persisted** on a new ticket row (tickets are stateless signed blobs today; to attribute
  egress by tag we must record the tags when egress happens).

### Current state (verified)

- `GetDownloadTicket` (`coord/file/endpoint.go:842`) authorizes the file owner, signs a 24-byte
  `DownloadTicketContent{FileUUID, ExpiresAt}` (`coord/file/download_ticket.go:20`) and returns it.
  Nothing is persisted.
- `GetDownloadInfoByTicket` (`coord/file/endpoint.go:690`) verifies signature + expiry, then
  `buildManifest(FileUUID)` (`:724`) assembles the manifest and calls
  `recordDownloadBandwidth(file.ContractId, usage)` (`:805`), which does
  `AddContractEgress(contractID, now, egress)` (`coord/db/accounting_repo.go:309`, best-effort/
  non-fatal) plus per-worker allocated (fatal).
- Egress is billed **per contract** only. There is no per-ticket / per-tag dimension anywhere,
  and no metadata is queryable/indexed today.
- Usage query surface = gRPC `AccountingService` (`coord/accounting/endpoint.go`,
  `GetClientUsage(contract_id, from, to) -> {storage_byte_hours, egress}`); ownership enforced by
  comparing `contract.OwnerId` to the mTLS caller (`endpoint.go:60-66`).
- Existing KV-metadata precedent to copy: `coord/file/metadata.go` (`Normalize()` lowercases keys,
  `Validate()` enforces a byte cap + control-char/CRLF guard, `MaxUserMetadataBytes = 2048`).

## Design

S3 cost-allocation-tags on the download-ticket path. Tags are stored **once** on a persisted
ticket row; egress is rolled up **per ticket**; tag queries JOIN the two and aggregate. This keeps
the rollup table bounded by ticket count (no per-(key,value) cardinality explosion) and keeps tags
mutable/manageable in one place. A GIN index on the tags column makes tag filters fast.

### Data model (2 new tables, new migration `coord/db/migrations/0000XX_download_tickets.up.sql`)

`download_tickets`
- `id UUID PK` — the ticket id (what the signed token now references)
- `file_id UUID` — file this ticket downloads (was carried in the token)
- `contract_id UUID` — denormalized from the file, for egress attribution + ownership scoping
- `owner_id UUID` — contract owner, for list/ownership checks
- `tags JSONB` — plaintext `map[string]string` (same treatment as `files.user_metadata`)
- `expires_at TIMESTAMPTZ`, `created_at TIMESTAMPTZ`
- `purge_at TIMESTAMPTZ` — GC deadline, computed at insert as `expires_at + (expires_at - created_at)`
  = retain the row for **2x the ticket lifetime** (one extra lifetime past expiry) so late-arriving
  downloads and usage queries still resolve, then it's deleted.
- indexes: GIN on `tags`, btree on `contract_id`, `owner_id`, `purge_at`

`ticket_egress_rollups` (mirrors `contract_egress_rollups`)
- `ticket_id UUID`, `interval_start TIMESTAMPTZ`, `egress_bytes BIGINT`, PK `(ticket_id, interval_start)`
- FK `ticket_id -> download_tickets(id) ON DELETE CASCADE` — so purging a ticket row also drops its
  egress detail in one shot (same idiom as `files -> segments ON DELETE CASCADE`). Durable billing
  stays intact because `contract_egress_rollups` is written independently and is never purged here.

### Token layout change

Reinterpret the existing 24-byte token as `TicketID(16) + ExpiresAt(8)` instead of
`FileUUID(16) + ExpiresAt(8)` — same size, still signed over both fields. `file_id` moves into the
persisted row. Expiry stays in the token so an expired token is rejected before any DB hit.
- **Breaking**: tokens issued before this change stop verifying. Acceptable pre-release; call it out.
- Edit `DownloadTicketContent` field `FileUUID -> TicketID` in `coord/file/download_ticket.go`
  (encode/decode/sign/verify layout is otherwise unchanged).

### Changes by file

1. **Proto** `pkg/pb/coord/file/v1/file.proto`: add `map<string,string> tags = 3;` to
   `GetDownloadTicketRequest` (`:277`). Regenerate `file.pb.go` (existing codegen path).
2. **Tag validation** new `coord/file/ticket_tags.go`: `TicketTags` type with `Normalize()` +
   `Validate()` copied from `metadata.go` style. S3-like limits: max 10 tags, key <=128 B,
   value <=256 B, control-char/CRLF guard. (Keys: keep case as-is for S3 fidelity — note this
   diverges from `metadata.go`'s lowercasing; flag for review.)
3. **Ticket row store** new `coord/file/download_ticket_store.go` (+ gorm model / `TableName()`):
   `CreateTicket(ctx, *DownloadTicketRow)` and `GetTicket(ctx, id)`. Follow existing
   `coord/file/store.go` interface + repo pattern.
4. **Issue path** `coord/file/endpoint.go` `GetDownloadTicket` (`:842`): after owner auth, resolve
   file -> contract_id + owner; validate `req.Tags`; `store.CreateTicket(row)`; sign token with the
   new `TicketID`. `coord/file/service.go` gains the resolve+create orchestration (mirror `Create`).
5. **Download path** `coord/file/endpoint.go` `GetDownloadInfoByTicket` (`:690`): after verify+expiry,
   `store.GetTicket(content.TicketID)` -> file_id, contract_id, ticket_id; pass ticket_id through
   `buildManifest`/`recordDownloadBandwidth`. Owner path `GetDownloadInfo` passes no ticket.
6. **Egress attribution** `recordDownloadBandwidth` (`:805`): add optional `ticketID *vo.UUID`; when
   set, also call new `AddTicketEgress(ticketID, now, egress)` (best-effort/non-fatal, same policy as
   contract egress). New `AddTicketEgress` + `ListTicketEgress` + `SumEgressByTag` in
   `coord/db/accounting_repo.go` next to `AddContractEgress`.
7. **Client-facing query RPCs — the deliverable clients call.** Extend the existing, already
   client-reachable `AccountingService` (`pkg/pb/coord/accounting/v1/accounting.proto`), which is
   registered for clients in `setupAccountingEndpoint` (`coord/peer.go:470-482`,
   `RegisterAccountingServiceServer`) and authed by mTLS peer identity + contract-owner check
   (`coord/accounting/endpoint.go:40,60`). Reuse that whole surface — no new service, no new server
   wiring — so the methods are exposed to clients the moment the stubs regenerate.

   Add two RPCs to the `service AccountingService` block and their messages, regenerate the pb, then
   implement the handlers in `coord/accounting/endpoint.go` (reusing `PeerIdentityFromContext` +
   the `contract.OwnerId.Equal(caller)` guard at `:60`) backed by new `UsageService` methods
   (`coord/accounting/usage.go`) and repo queries (`coord/db/accounting_repo.go`):

   - `rpc GetTicketUsage(GetTicketUsageRequest) returns (UsageReport)` — `{ticket_id, from, to}` ->
     one ticket's summed egress. Owner check: load the ticket, verify `owner_id == caller`.
   - `rpc GetUsageByTag(GetUsageByTagRequest) returns (UsageByTagResponse)` —
     `{contract_id, tag_key, optional tag_value, from, to}`. Owner check on the contract (`:60`
     pattern). Key only -> group by value (S3 "group by tag key"), returns
     `repeated { string tag_value; int64 egress; }`; key+value -> single filtered sum. Repo SQL joins
     `ticket_egress_rollups` -> `download_tickets` with `tags ? :key` (GIN), `contract_id = :cid`,
     `interval_start BETWEEN :from AND :to`, `GROUP BY tags->>:key`.

   Both return egress in the existing `UsageReport.egress` field / a small new response message;
   `storage_byte_hours` stays 0 for ticket queries (egress-only, by design).

### Retention / GC chore (tickets are high-volume — must be purged)

New chore package `coord/file/ticketcleanup/` copied from `coord/file/expireddeletion/chore.go`
(same `Config{Interval 1h, Enabled true, ListLimit 100}`, `sync2.NewCycle`, batched loop, log-don't-
fail-the-coord policy). It deletes rows whose retention window has elapsed:
- Store method `DeleteTicketsPastPurge(ctx, now, batchSize) (int64, error)` -> `DELETE FROM
  download_tickets WHERE purge_at < :now LIMIT :batch` (loop until a short batch, like
  `deleteExpiredFiles`). The `ticket_egress_rollups` rows go with it via ON DELETE CASCADE.
- Unlike `expireddeletion` (fixed `uploadWindow`), the cutoff is **per-row** (`purge_at`, = 2x each
  ticket's own lifetime), so the chore only needs `now()` — no window arg.
- Wire in `coord/peer.go` (`setupTicketCleanup`, mirror `setupExpiredDeletion` at `:453` + the
  lifecycle item at `:461`; add field near `:811`; construct near `:915`) and add
  `TicketCleanup ticketcleanup.Config` to `coord/config.go` (mirror `ExpiredDeletion` at `:33`).

### Out of scope (follow-up track)

- **Client plumbing**: setting `GetDownloadTicketRequest.tags` from uplink/edge/go-sdk. The go-sdk
  lives in `../go-sdk` (separate repo, not in this worktree), and real clients don't even populate
  file metadata yet (`internal/testplanet/edgeserver_metadata_test.go` stamps it directly). Coord
  side is fully testable via gRPC/testplanet regardless.
- Tag encryption (existing file metadata is plaintext too — match that).
- Ticket revocation / listing / TTL cleanup of `download_tickets` rows (add later; note a GC chore
  will eventually be wanted, like the committed-TTL cleanup already tracked elsewhere).

## Verification (end-to-end, Postgres-backed)

Reproduce as close to the real download flow as possible, in `internal/testplanet` (pattern:
`edgeserver_metadata_test.go`, which builds a real coord DB + overlay/order/accounting):

1. Create + commit a file (existing helpers).
2. `GetDownloadTicket(file_id, expires_at, tags={env:prod, team:ml})` -> assert a `download_tickets`
   row exists with those tags, and the returned token verifies to its `TicketID`.
3. `GetDownloadInfoByTicket(ticket)` -> assert manifest built AND a `ticket_egress_rollups` row
   accumulated egress == Σ segment encrypted sizes (matches `contract_egress_rollups`).
4. `GetTicketUsage(ticket_id, from, to)` -> egress equals step 3.
5. Two tickets with `env=prod` and one with `env=dev`; `GetUsageByTag(contract_id, "env", from, to)`
   -> per-value breakdown `{prod: sum, dev: sum}`; with `tag_value="prod"` -> just the prod sum.
6. Ownership: a different identity querying another owner's ticket/contract -> `PermissionDenied`.
7. GC: set the cleanup chore's `SetNow` past a ticket's `purge_at` (2x its lifetime), run one loop
   -> assert the `download_tickets` row AND its `ticket_egress_rollups` rows are gone (cascade), while
   `contract_egress_rollups` is untouched.

Unit tests: token encode/decode round-trip (`TicketID`), `TicketTags.Validate` limits,
`AddTicketEgress` upsert-add, and `purge_at = 2*expires_at - created_at` computed correctly at insert.

Commands: `go test ./coord/... ./internal/testplanet/...` (DB tests need the coord Postgres, as the
existing accounting/file tests already do). Then run the coord binary against a local cluster and
exercise the ticket -> download -> `GetUsageByTag` path over gRPC to confirm real behavior.
