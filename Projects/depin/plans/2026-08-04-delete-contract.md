# Plan: DeleteContract (contract = S3 bucket equivalent)

## Context

`coord/storage` manages **contracts** - the per-client record a file must live under (the
bucket equivalent). Today `StorageService` has exactly one RPC, `CreateContract`
(`pkg/pb/coord/storage/v1/storage.proto:10-12`), and `storage.ContractStore` has exactly two
methods, `Create` + `GetByID` (`coord/storage/contract_store.go:9-12`). **A contract can be
created but never removed.** Clients accumulate dead contracts forever, and the file-level
counterpart (`FileService.DeleteFile`, shipped) has no bucket-level sibling.

The schema has been waiting for this: `contracts.deleted_at` exists with a partial index
(`coord/db/migrations/000002_init_client.up.sql:34,50`), `ContractStatus.DELETED = 2` exists in
`pkg/pb/coord/contract/v1/contract.proto:19`, and both are **entirely unread** today. More
pointedly, `coord/db/accounting_repo.go:681-683` already documents the intended semantics:

> `// Deleted contracts are deliberately not filtered out. Usage accrued before a`
> `// contract was deleted is still owed, and dropping the row would silently forgive`
> `// it. (Nothing deletes contracts today, but the invoice should not depend on that.)`

Outcome: a client can delete a storage contract; its files and segments are removed from the
coordinator (pieces reclaimed lazily by GC, exactly as `DeleteFile` does); its billing and usage
history survive intact.

### Confirmed design decisions

| Decision | Choice |
|---|---|
| Contract row | **Soft delete** - set `deleted_at` + `status='DELETED'`, keep the row |
| Files / segments | **Hard delete**, batched, explicit (not via FK cascade) |
| Non-empty contract | **Reject** with `FailedPrecondition` unless `force=true` (Storj `DeleteBucket` parity) |
| Drain execution | **Synchronous batched loop** inside the RPC, no new chore |
| Client surface | Coord RPC **+ go-sdk wrappers** (`DeleteContract`, and `DeleteFile` which is also missing) |

### Why soft delete is load-bearing

`contract_storage_tallies`, `contract_egress_rollups` and `client_invoices` all carry
`contract_id` with **no foreign key** (`000010:5-16`, `000015:7-17`, `000037:20-51`). A hard
`DELETE FROM contracts` would:

- orphan every one of those rows, and
- silently drop the contract's unbilled usage - `QueryContractPeriodUsage`
  (`coord/db/accounting_repo.go:684-765`) is *driven by* `Table("contracts")` at `:693-696`, so a
  vanished row means no invoice line, and
- be unrecoverable: the charge idempotency key is `(owner_id, period)`
  (`000037_client_billing.up.sql:83-85`), so re-running `record-period` after the fact is a no-op.

Soft delete keeps all of that correct with zero changes to the accounting or billing code.

### Why files are hard-deleted explicitly, not by cascade

`files.contract_id -> contracts(id) ON DELETE CASCADE` is the only surviving FK to `contracts`,
and it fans out three levels unbatched (`files -> segments -> segment_piece_uploads`). The
codebase already rejects that approach where it means to delete: both `DeleteFile`
(`coord/db/file_repo.go:99-130`) and `DeleteExpiredPendingFiles` (`:139-171`) delete segments
explicitly, with the reasoning at `:132-138`. We follow the same rule, plus `ORDER BY id LIMIT ?`
batching.

---

## Implementation

### 1. Proto - `pkg/pb/coord/storage/v1/storage.proto`

Add one RPC to the existing `StorageService` and two messages. Follow the file's existing
gogoproto idiom (`customtype` + `nullable=false` for UUIDs), matching `DeleteFileRequest`
(`pkg/pb/coord/file/v1/file.proto:272-275`).

```proto
service StorageService {
  rpc CreateContract(CreateContractRequest) returns (CreateContractResponse);
  rpc DeleteContract(DeleteContractRequest) returns (DeleteContractResponse);
}

message DeleteContractRequest {
  bytes contract_id = 1 [
    (gogoproto.customtype) = "aioz-depin/internal/vo.UUID",
    (gogoproto.nullable) = false
  ];
  // force first deletes every file under the contract. Without it, a contract
  // that still holds files is rejected with FailedPrecondition. Mirrors storj
  // BucketDeleteRequest.delete_all.
  bool force = 2;
}

message DeleteContractResponse {
  int64 deleted_file_count = 1;
  int64 deleted_segment_count = 2;
}
```

Regenerate with `make proto` (= `buf generate`, `Makefile.build:252-254`). No `coord/peer.go`
change is needed - `storagepb.RegisterStorageServiceServer` at `coord/peer.go:321` already wires
every method on the service.

### 2. Store interfaces - `coord/storage/contract_store.go`

Extend `ContractStore`, and add a **new consumer-side `FileStore` interface declared in
`coord/storage`**:

```go
type ContractStore interface {
	Create(context.Context, *Contract) error
	// GetByID returns a live contract. Soft-deleted contracts are invisible to
	// it, so every write path (CreateFile, upload) rejects them for free.
	GetByID(context.Context, vo.UUID) (*Contract, error)
	// GetByIDIncludingDeleted also returns soft-deleted contracts. Used by
	// deletion (so a retry is idempotent) and by usage reads, whose whole point
	// is that history outlives the contract.
	GetByIDIncludingDeleted(context.Context, vo.UUID) (*Contract, error)
	// SoftDelete stamps deleted_at + status='DELETED'. A no-op on an already
	// deleted contract.
	SoftDelete(ctx context.Context, id vo.UUID, now time.Time) error
}

// FileStore is the narrow slice of file persistence that contract deletion
// needs. It is declared HERE rather than imported from coord/file because
// coord/file already imports coord/storage - the same directed-edge constraint
// that forced coord/tags into its own leaf package (see coord/tags/tags.go:1-6).
// Using only context/vo types keeps this cycle-free.
type FileStore interface {
	ContractHasFiles(ctx context.Context, contractID vo.UUID) (bool, error)
	DeleteFilesByContract(
		ctx context.Context, contractID vo.UUID, batchSize int,
	) (deletedFiles, deletedSegments int64, err error)
}
```

### 3. Persistence - `coord/db/contract_repo.go` + `coord/db/file_repo.go`

`contractRepository` (add `deleted_at IS NULL` to the existing `GetByID`, plus the two new
methods). `SoftDelete` writes `deleted_at`, `updated_at` and `status = 'DELETED'`, scoped
`WHERE id = ? AND deleted_at IS NULL`.

`fileRepository` gains two methods. `ContractHasFiles` uses `SELECT EXISTS(...)` (not `COUNT(*)`)
so the emptiness check is O(1) on `idx_files_contract_id` regardless of file count.
`DeleteFilesByContract` copies the `DeleteExpiredPendingFiles` CTE shape verbatim
(`coord/db/file_repo.go:150-161`), swapping the selection predicate:

```sql
WITH doomed AS (
	SELECT id FROM files
	WHERE contract_id = ?
	ORDER BY id
	LIMIT ?
), deleted_files AS (
	DELETE FROM files WHERE id IN (SELECT id FROM doomed) RETURNING id
), deleted_segments AS (
	DELETE FROM segments WHERE file_id IN (SELECT id FROM deleted_files) RETURNING file_id
)
SELECT (SELECT COUNT(*) FROM deleted_files), (SELECT COUNT(*) FROM deleted_segments)
```

`segment_piece_uploads` rows for in-flight uploads clear via their existing
`ON DELETE CASCADE` to `segments` (`000003_init_upload.up.sql:95-99`).

Expose `FileStore` from the UoW - `coord/db/uow.go` already lazily builds `fileRepository`
alongside the others; add a `FileStoreForContracts()` (or reuse the existing file repository
accessor) returning `storage.FileStore`.

### 4. Usecase - `coord/storage/service.go`

`StorageUsecase` gains a `files storage.FileStore` dependency and one method. **Order is
mark-then-drain**: stamping `deleted_at` first makes `GetByID` reject the contract immediately, so
no new upload can land files behind the drain. A failed drain leaves a soft-deleted contract with
residual files, and re-invoking `DeleteContract` resumes it - which is why authorization loads the
contract *including* deleted ones.

```go
var ErrContractNotEmpty = StorageError.New("contract is not empty")

// deleteBatchSize bounds one drain statement. Mirrors storj metabase's
// deleteBatchSizeLimit (50) and coord/file/expireddeletion's ListLimit (100).
// Promote to Config if a deployment ever needs to tune it.
const deleteBatchSize = 100

func (s *StorageUsecase) DeleteContract(
	ctx context.Context, contractID, caller vo.UUID, force bool,
) (deletedFiles, deletedSegments int64, err error)
```

Steps:
1. `contractRepo.GetByIDIncludingDeleted` -> `vo.ErrRecordNotFound` maps to `vo.NotFound`.
2. `!contract.OwnerId.Equal(caller)` -> `vo.Forbidden`. (Same shape as
   `coord/file/service.go:143` and `coord/accounting/endpoint.go:60`.)
3. If `!force`: `files.ContractHasFiles(...)`; true -> `ErrContractNotEmpty`, return without
   touching anything.
4. `contractRepo.SoftDelete(ctx, id, time.Now())`.
5. If `force`: loop `files.DeleteFilesByContract(id, deleteBatchSize)`, accumulating both counts,
   until a batch returns fewer files than `deleteBatchSize`.
6. Return the totals.

Steps 3+4 run inside `WithTx` so the emptiness check and the mark commit together.

### 5. Endpoint - `coord/storage/endpoint.go`

Mirror `CreateContract`'s shape: `identity.PeerIdentityFromContext(ctx)` for the caller, then map
errors. Do **not** route the two interesting failures through `vo.GrpcError` - that helper has no
`Forbidden` case at all (`internal/vo/errors.go:64-76`), so it would return `codes.Internal`.
Use the direct `grpcerr.NamedError` pattern already established at
`coord/accounting/endpoint.go:61-65`:

- owner mismatch -> `grpcerr.NamedError("forbidden", codes.PermissionDenied, "forbidden")`
- `ErrContractNotEmpty` -> `grpcerr.NamedError("contract-not-empty", codes.FailedPrecondition, ...)`
- everything else -> `vo.GrpcError(err)` (`NotFound` maps correctly)

Log at Debug with `contract_id`, `force`, `deleted_file_count`, mirroring
`coord/file/endpoint.go:689-695`.

### 6. Keep usage history readable - `coord/accounting/endpoint.go:56`

`GetClientUsage` with an explicit `contract_id` gates on `e.contracts.GetByID`. Once `GetByID`
filters soft-deleted rows, an owner could no longer read the usage history of a contract they
just deleted. Switch **this one call site** to `GetByIDIncludingDeleted`. The ownership check
below it is unchanged, and the tag-usage and period-usage queries need no change at all - they
join `contracts` and the row is still there.

Leave `coord/file/service.go:77` and `:135` on the filtered `GetByID`: rejecting uploads and
downloads under a deleted contract is the desired behaviour.

### 7. Migration `000039_contract_status_backfill` (small)

`NewContract` (`coord/storage/contract.go:13-32`) never sets `Status`, so every existing contract
has `status = ''` in a `TEXT NOT NULL` column. Writing `'DELETED'` on delete only means something
if live rows say something too:

- up: `UPDATE contracts SET status = 'ACTIVE' WHERE status = '' OR status IS NULL;`
- down: the inverse.
- Set `Status: "ACTIVE"` in `NewContract` so new rows are consistent.

`contracts.deleted_at` and its partial index already exist - no schema change needed for the
delete itself. Create with `make -f Makefile.migration ...` (`MIGRATE_CREATE_FLAGS = -seq -digits 6`);
next number is **000039** (highest today is `000038_wallet_sweeps`).

### 8. go-sdk (sibling repo `/home/tuan/work/depin-workspace/go-sdk`)

Add to the SDK's public surface, next to `CreateContractWithPlacement` (`placement.go:32`):

- `func (c *Client) DeleteContract(ctx context.Context, contractID vo.UUID, force bool) (deletedFiles int64, err error)`
- `func (c *Client) DeleteFile(ctx context.Context, fileID vo.UUID) error` - missing today even
  though the coord RPC has shipped; add it so the e2e test can set up and assert realistically.

Proto resync on the SDK side is targeted (copy the regenerated `storage.pb.go` /
`storage_grpc.pb.go`), not the wipe-all sync script.

**Build caveat:** `go.mod:402` replaces `aioz-depin/go-sdk` with a *pinned pseudo-version*, not
the local checkout. For development, temporarily point it at the symlink already present at
`/home/tuan/.treehouse/depin-b971d9/2/go-sdk`:
`replace aioz-depin/go-sdk => ../go-sdk`. Restore the pinned form before finishing, after the
go-sdk change is committed and pushed.

---

## Critical files

| File | Change |
|---|---|
| `pkg/pb/coord/storage/v1/storage.proto` | new RPC + 2 messages |
| `coord/storage/contract_store.go` | extend `ContractStore`, add `FileStore` |
| `coord/storage/service.go` | `DeleteContract` usecase, `ErrContractNotEmpty` |
| `coord/storage/endpoint.go` | `DeleteContract` handler |
| `coord/storage/contract.go` | `NewContract` sets `Status: "ACTIVE"` |
| `coord/db/contract_repo.go` | `deleted_at IS NULL` on `GetByID`, `GetByIDIncludingDeleted`, `SoftDelete` |
| `coord/db/file_repo.go` | `ContractHasFiles`, `DeleteFilesByContract` CTE |
| `coord/db/uow.go` | expose `storage.FileStore` |
| `coord/peer.go:313-323` | inject the file store into `setupStorage` |
| `coord/accounting/endpoint.go:56` | use `GetByIDIncludingDeleted` |
| `coord/db/migrations/000039_*` | status backfill |
| `coord/storage/README.md` | document delete (note: lines 11-15/30-34 are already stale re: the removed `StorageConfig`) |
| `../go-sdk/` | `DeleteContract` + `DeleteFile` wrappers |

Existing code to reuse, not reinvent: the `DeleteExpiredPendingFiles` CTE
(`coord/db/file_repo.go:139-171`) as the drain template; `identity.PeerIdentityFromContext` for
the caller; `grpcerr.NamedError` for `PermissionDenied`/`FailedPrecondition`;
`coorddbtest.Run` for DB tests; `internal/testplanet` for e2e.

---

## Verification

**1. Unit - `coord/storage/service_test.go` (new; the package has no tests today).** Fakes for
`ContractStore` and `FileStore`, following `coord/file/service_test.go`'s `fakeStore` /
`fakeContractStore` style (call counters + canned results):

- empty contract, `force=false` -> soft-deleted, no drain call
- non-empty, `force=false` -> `ErrContractNotEmpty`, `SoftDelete` **not** called
- non-empty, `force=true` -> drain loops until a short batch (script `100, 100, 37`), counts summed, contract marked
- wrong caller -> `vo.Forbidden`, nothing mutated
- unknown contract -> `vo.NotFound`
- already soft-deleted with residual files + `force=true` -> resumes the drain (idempotent retry)

**2. DB integration - `coord/db/contract_repo_test.go` (new).** `coorddbtest.Run`, real Postgres
(skips without `DEPIN_TEST_POSTGRES`; container on `:5445` per `coorddbtest.DefaultPostgres`).
Note `const testPlacementID = 999` is required for `segments.placement_id`'s FK, as in
`coord/db/file_repo_test.go:20`.

- `SoftDelete` sets `deleted_at`/`status`; `GetByID` then returns not-found while
  `GetByIDIncludingDeleted` returns the row
- `ContractHasFiles` true/false
- `DeleteFilesByContract` with 3 files (each with segments) and `batchSize=2` -> two calls, correct
  file/segment counts, rows gone, **a second contract's files untouched**

**3. e2e - `internal/testplanet/delete_contract_test.go` (new).** Model on
`internal/testplanet/tag_usage_test.go` (which already creates contracts via
`coord.Storage.Endpoint.CreateContract` and runs real uploads):

1. create contract, upload a real file, run a tally snapshot
2. `DeleteContract{force:false}` -> `codes.FailedPrecondition`
3. `DeleteContract{force:true}` -> `deleted_file_count == 1`
4. assert `files` and `segments` rows for that contract are gone
5. assert the `contracts` row is **still present** with `deleted_at` set
6. assert `GetClientUsage(contract_id)` still returns the pre-delete usage, and
   `QueryContractPeriodUsage` still yields a line for the contract - the billing invariant
7. assert `CreateFile` against the deleted contract now fails

**Commands**

```sh
make proto
go build ./... && go vet ./coord/... 
go test ./coord/storage/... ./coord/file/... ./coord/db/...
export DEPIN_TEST_POSTGRES='postgresql://admin:admin123@localhost:5445/hub?sslmode=disable'
go test ./coord/db/... -run 'Contract'
go test ./internal/testplanet/... -run 'DeleteContract' -v
go test ./coord/... ./internal/testplanet/...   # full suite, no regressions
```

---

## Known gaps, stated explicitly (not fixed here)

- **Pieces are not actually reclaimed on a default deployment.** Both GC stages default to
  `Enabled: false` (`coord/gc/bloomfilter/config.go:18`, `coord/gc/sender/config.go:13`) and
  `bloomfilter.Storage` has only an in-process `MemStorage`. Deleted files' pieces stay on workers
  until GC is turned on. This is pre-existing and identical for `DeleteFile`.
- **`vo.GrpcError` has no `Forbidden` case** (`internal/vo/errors.go:64-76`), so
  `DeleteFile`'s wrong-owner path returns `codes.Internal` instead of `PermissionDenied`. This plan
  routes around it in the endpoint rather than changing shared error mapping; fixing
  `vo.GrpcError` is a separate, worthwhile change.
- **Mid-upload deletion** loses worker-side allocated bandwidth with no settlement path (pieces are
  already written when `CommitSegment` finds the file gone). Storj has the same race on
  `DeleteBucket`; accepted.
- **Dangling queue rows** in `repair_queue` / `audit_queue` / `reverification_queue` (no FK to
  segments) - verified harmless: the repairer drops missing segments
  (`coord/repair/repairer/segments.go:102`) and audit handles `vo.ErrRecordNotFound`
  (`coord/audit/worker.go:108`, `reverifyworker.go:122`).
- **No `ListContracts`** exists, so a client cannot enumerate what it owns to decide what to
  delete. Out of scope.
