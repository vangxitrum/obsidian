---
type: decision
tags: [depin, coord, storage, contract, delete]
created: 2026-08-05
agent: main
---

Implemented DeleteContract per plan `projects/depin/plans/2026-08-04-delete-contract.md`, on worktree
`/home/tuan/.treehouse/depin-b971d9/2/depin`, branch `feat/delete-contract`. Uncommitted — awaiting
user review (global CLAUDE.md rule: never commit unless explicitly asked).

Shape: soft-delete the contract row (`deleted_at` + `status=DELETED`, so billing/usage history
survives — `contract_storage_tallies`/`contract_egress_rollups`/`client_invoices` have no FK to
`contracts`), hard-delete files/segments explicitly+batched (same CTE shape as
`DeleteExpiredPendingFiles`), mark-then-drain ordering so `CreateFile`'s live-only `GetByID` rejects
uploads immediately. `!force` on a non-empty contract → `FailedPrecondition`, nothing touched.

**Plan deviation (user-approved, not silent):** plan section 2 specced `ContractStore`/`FileStore`
with no `WithTx`, but section 4 required "steps 3+4 (emptiness-check + soft-delete) run inside
WithTx" — a real gap, since those are two different interfaces. Stopped and asked; user picked
"new combined transaction dependency" over the two smaller options. Added
`storage.TxRunner.WithTx(ctx, fn func(ContractStore, FileStore) error) error`, backed by
`*db.UnitOfWork.StorageTxRunner()` (thin adapter, `coord/db/uow.go`) so both stores share one real
Postgres transaction. `NewStorageUsecase` gained two params (`files FileStore`, `txRunner TxRunner`)
beyond what the plan's constructor signature showed.

**Plan bug found+fixed without stopping (mechanical, one correct answer):** plan pseudocode used
`StorageError.New("contract is not empty")` — `errs.Tag` (zeebo/errs/v2) has no `.New` method (only
`.Wrap`/`.Errorf`/`.Error`/`.Name`). Fixed to the codebase's own established sentinel-error idiom
(`coord/vo/errors.go`'s style): `var ErrContractNotEmpty = errors.New(...)`, wrapped via
`StorageError.Wrap(...)` where returned — same shape as `vo.NotFound`/`vo.Forbidden` usage elsewhere.

Go-sdk sibling repo (`/home/tuan/work/depin-workspace/go-sdk`, uncommitted): added
`Client.DeleteContract` (placement.go, next to `CreateContractWithPlacement`) and `Client.DeleteFile`
(upload.go, was missing despite the coord RPC already shipping). Proto resync was **exactly**
line-for-line identical to depin's regenerated `storage.pb.go`/`storage_grpc.pb.go` modulo one
import-path substitution (`aioz-depin/internal/vo` → `aioz-depin/go-sdk/internal/vo`) — confirmed via
diff before copying, so future targeted resyncs can just `sed` + copy rather than hand-porting.

depin's `go.mod` `replace aioz-depin/go-sdk` is currently pointed at the local `../go-sdk` symlink
(dev-only, per the plan's own build caveat) — **must be restored to the pinned
`gitlab.internal/aioz-depin/go-sdk` pseudo-version** once the go-sdk change is committed+pushed. Not
done yet since nothing was committed this session.

Verification: unit tests (`coord/storage/service_test.go`, 6 cases incl. idempotent-retry-resumes-
drain), DB integration (`coord/db/contract_repo_test.go` against real Postgres on `:5445`), e2e
(`internal/testplanet/delete_contract_test.go` — full lifecycle incl. billing-survives-delete
invariant) all green. Full `go test ./coord/... ./internal/testplanet/...` green except two confirmed
pre-existing/unrelated failures: `TestEdgeserverMetadataHeaders` (already documented, see
[[edgeserver-metadata-headers]]) and `TestSettlementRejectsOverAllocation` (confirmed flaky via 3x
isolated reruns — passed once, failed twice, no contract/storage code involved).

**Gotcha:** this session's root filesystem hit 100% full (217M free) mid-verification, which broke
`go test`'s linker for ANY package needing a fresh test binary link (not just mine) — misleading
because the failures spanned totally unrelated packages (`coord/gc`, `coord/repair`, `coord/tags`,
etc.). Diagnosed by checking `df -h /`; user freed space (down to 92% used) and the full suite passed
clean afterward.

Related: [[contract-file-tags-usage]], [[delete-file-phase1]], [[edgeserver-metadata-headers]]
