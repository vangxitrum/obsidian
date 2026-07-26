---
type: fact
tags: [depin, download-ticket, usage-accounting, s3-tags, coord, superseded]
created: 2026-07-13
agent: main
---

**SUPERSEDED 2026-07-14** — see [[contract-file-tags-usage]]. This whole design
was reverted: checking it against real S3 semantics showed S3 has no equivalent to
a tagged presigned URL (presigned URLs are signed client-side, never a server-minted
taggable resource). S3's actual model is contract tags (bucket-equivalent) + file
tags (object-equivalent). Kept below for historical context and reusable gotchas
(jsonb GROUP BY placeholder collision, egress-billing basis, go-sdk symlink) — do
not build on the design itself.

Implemented S3-style cost-allocation tags on download tickets + tag-based usage
queries, on branch `feat/ticket-tags-usage` (uncommitted at end of session — user
said "don't commit unless explicitly asked").

**Design**: entity = the presigned download ticket (`coord/file/download_ticket.go`),
not file/contract. Usage = egress only (a ticket has no storage lifecycle). Tags
persisted (tickets were previously stateless signed blobs with no DB row).

**Key pieces**:
- Migration `000032_download_tickets` — new `download_tickets` (id, file_id,
  contract_id, owner_id, tags JSONB w/ GIN index, expires_at, created_at, purge_at)
  + `ticket_egress_rollups` (mirrors `contract_egress_rollups`, FK ON DELETE CASCADE
  to download_tickets).
- Token layout changed: `DownloadTicketContent.FileUUID` renamed to `TicketID` —
  the 24-byte signed token now carries the ticket row id, not the file id directly
  (file_id moved into the persisted row). Breaking for any tokens issued before this
  change (fine pre-release).
- `purge_at = expires_at + (expires_at - created_at)` — 2x the ticket's own
  lifetime — computed once in `file.NewDownloadTicketRow`. New GC chore
  `coord/file/ticketcleanup` (mirrors `coord/file/expireddeletion`'s
  Config/Chore/batched-loop shape) purges past-due rows; `ticket_egress_rollups`
  rows go with them via the FK cascade, `contract_egress_rollups` (durable billing)
  is untouched.
- New client-facing RPCs on the *existing* `AccountingService`:
  `GetTicketUsage(ticket_id, from, to)` and
  `GetUsageByTag(contract_id, tag_key, tag_value?, from, to)` (empty tag_value =
  grouped by every value seen; non-empty = single filtered sum).
- `coord/accounting/endpoint.go` gained a `TicketStore` dependency (narrow
  interface satisfied by `coord/file`'s real file store — `coorddb.NewFileStore`,
  wired via `setupAccountingEndpoint` in `coord/peer.go`) purely for
  `GetTicketUsage`'s ownership check (load ticket, verify `owner_id == caller`).

**Real bugs caught by testing** (both fixed):
1. Postgres SQL: `SELECT tags->>?  ... GROUP BY tags->>?` (same tagKey value bound
   twice, via two separate `?` placeholders) fails
   `"column must appear in GROUP BY"` — Postgres treats each parameter placeholder
   as a distinct unknown at parse time, even if bound to an identical runtime value.
   Fix: `GROUP BY 1` (ordinal reference to the SELECT list's first column).
2. Egress is billed as the segment's post-encryption/erasure-padded
   `EncryptedSize` (`coord/file/endpoint.go`'s `buildDownloadSegment`), NOT the
   plaintext upload size — a naive e2e test asserting egress == len(uploaded bytes)
   fails; must sum `segments.encrypted_size` for the real expected value.

**Known design gap, explicitly accepted** (not fixed): `GetDownloadTicketResponse`
never returns the ticket's plain `ticket_id` — only the opaque signed token, which
ordinary clients cannot decode (only the coordinator's signer can verify/decode it).
So `GetTicketUsage(ticket_id)` is unreachable by a client holding just the token;
it's only usable by something with server-side DB knowledge of ticket_id (e.g. a
future admin/list-tickets tool). Decided to leave as-is after discussing the S3
analogy: **S3 has no "usage for one presigned URL" API at all** — presigned URLs
are signed client-side and never get a server-minted opaque ID; S3's cost
allocation tags live on buckets/objects (identifiers the client always already
knows), and Cost Explorer/Cost & Usage Reports are the only tag-based usage query
surface. So `GetUsageByTag(contract_id, tag)` is the real, S3-faithful
client-facing query path; `GetTicketUsage` is a lower-level primitive with no S3
equivalent, not a bug to fix.

**Environment gotcha**: this treehouse worktree
(`/home/tuan/.treehouse/depin-b971d9/1/depin`) didn't have a sibling `../go-sdk`
checkout (the real repo's sibling at `/home/tuan/work/depin-workspace/go-sdk`),
so `internal/testplanet` (and anything importing `aioz-depin/go-sdk`) failed to
build until symlinked in:
`ln -s /home/tuan/work/depin-workspace/go-sdk /home/tuan/.treehouse/depin-b971d9/1/go-sdk`.
Worth checking for in future treehouse-worktree sessions on this repo.

**Verification**: full DB-integration test coverage against the live `coord-db`
Postgres container (`localhost:5445`, `DEPIN_TEST_POSTGRES` env var) —
`coord/db/ticket_repo_test.go` (CRUD round-trip, egress upsert, tag
grouping/filtering, GC purge+cascade) — plus pure-Go unit tests
(`coord/file/ticket_tags_test.go`, `download_ticket_store_test.go`,
`ticketcleanup/chore_test.go`) and a full real end-to-end test
(`internal/testplanet/ticket_usage_test.go` — real erasure-coded upload/download,
real gRPC endpoints, ownership rejection). Entire `coord/...` + `internal/testplanet`
suite green, zero regressions.
