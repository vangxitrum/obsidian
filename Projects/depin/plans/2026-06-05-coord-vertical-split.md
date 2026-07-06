# Coord Vertical Split Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **⚠️ COMMIT POLICY:** Commits require the user's standing approval for the refactor branch only (granted for the worker phase; re-confirm at execution start). Never push.

**Goal:** Restructure `coord/` from DDD layers into vertical subsystems (`contact/`, `storage/`, `file/`, `placement/`, `client/`, `wallet/`), each holding Endpoint/Service/Store, wired by a hand-written Peer; the UnitOfWork dissolves into per-subsystem Store interfaces implemented by a central `coord/db` package.

**Architecture:** Per `docs/superpowers/specs/2026-06-04-vertical-split-restructure-design.md`, executed like the completed worker phase (commits f0a3f8b..8f2731d): subsystem-by-subsystem, build + tests green per task, type names preserved unless documented. The one structural change beyond moves: usecases stop taking `repository.UnitOfWork` and instead take narrow Store interfaces (+ a `WithTx` method on stores that need transactions); `coord/db` implements all stores on top of the existing GORM repositories, which move there unchanged.

**Tech Stack:** Go, GORM, gRPC; module `aioz-depin`. No new dependencies.

**Baseline (verified 2026-06-05 on feat/worker-contact, post worker merge):** `go build ./coord/... ./cmd/coord/...` passes; `go test ./coord/...` passes (tests exist only for usecase/file, pkg/aes, pkg/formatter; integration test is env-gated). Pre-existing gofmt failure: `cmd/coord/main.go`. The gate below must stay fully green — coord has NO pre-existing test-failure allowance.

**Differences from the spec's first draft (planning findings):**
1. Subsystem order is dependency-driven: `wallet → client → storage → contact → placement → file` (contact owns the Worker store consumed by placement and file; spec's draft order had contact last).
2. `coord/domain/vo/errors.go` (cross-subsystem error values) → `coord/vo/` (tiny component-root shared package; the spec didn't assign it).
3. `coord/presentation/api/http/` (router + healthcheck) → `coord/server/` (it's HTTP server plumbing, not a subsystem).
4. Orphaned-but-thematic types move with their theme, never deleted (reward_* → wallet; audit/reservoir → placement; transferlog/object/task → file): they are product stubs, not verified-dead duplicates.
5. `cmd/coord/main.go` imports `coord/domain/entity` + `coord/infrastructure/repository` directly — those get fixed in the tasks that move their targets.

---

## Conventions used by every task

```bash
rewrite() { grep -rl "$1" --include='*.go' . | xargs -r sed -i "s|$1|$2|g"; }
```

**Gate** (after every task; must be fully green):

```bash
go build ./coord/... ./cmd/coord/... && \
go vet ./coord/... ./cmd/coord/... && \
go test ./coord/... && \
gofmt -l coord/ cmd/coord/ | grep -v 'cmd/coord/main.go' | (! grep .)
```

(`cmd/coord/main.go` fails gofmt pre-existing; everything else must be clean. `gofmt -w` every file you sed-edit — in-place path rewrites break import-group sorting.)

**Shared package-level files:** `coord/application/usecase/monkit.go` (and any shared `var mon`/helper file in `presentation/api/grpc/`) serve every file in their package. As subsystem tasks drain those packages: each NEW subsystem package that uses `mon.` needs its own `var mon = monkit.Package()` (worker phase precedent); when the LAST consumer leaves a package, absorb whatever the leavers still need and `git rm` the shared file. Check at every subsystem task: `grep -n 'mon\.' <moved files>`.

**Move/merge patterns:** same as the worker plan — `git mv` + package-clause sed + repo-wide qualifier `rewrite`; drop self-imports/self-qualifiers in moved files; the build lists unqualified same-package stragglers, fix by hand. Worker-phase lessons that apply here: gofmt every sed-touched file; broad qualifier rewrites (`usecase\.`, `entity\.`) must be checked for over-match before running (`grep` first); `worker/` and `uplink-sdk/` are OUT OF SCOPE — if a rewrite touches them, revert those hunks.

---

### Task 0: Preflight — workspace and branch

- [ ] **Step 1: Main-tree hygiene check.** `git -C /home/tuan/work/depin-workspace/depin status --porcelain` — the user has uncommitted edits (`dev/worker1/.air.toml`, `worker/retain/retain.go` formatting, deleted `.claude/worktrees/...` gitlink). ⚠️ EnterWorktree migrated uncommitted changes into the worktree during the worker phase and nearly lost them. The controller must get these committed/stashed by the user, or verify after worktree creation that they remain in the main tree, before any task runs.
- [ ] **Step 2: Create isolated worktree** on branch `refactor/coord-vertical-split` based on `feat/worker-contact` (EnterWorktree, then verify base: `git merge-base --is-ancestor d4fa940 HEAD`; if the harness based it on origin/main, `git reset --hard feat/worker-contact && git branch -m refactor/coord-vertical-split`).
- [ ] **Step 3: Record baseline** in the worktree: run the Gate; everything must pass. `go test ./coord/... 2>&1 | tee /tmp/coord-test-baseline.txt`.

### Task 1: coord/pkg/ratelimiter + http plumbing into server

**Files:**
- Move: `coord/utils/ratelimiter/` → `coord/pkg/ratelimiter/` (package unchanged); delete emptied `coord/utils/`
- Move: `coord/infrastructure/server/` → `coord/server/`
- Move: `coord/presentation/api/http/routes_http.go` → `coord/server/http_routes.go`; `coord/presentation/api/http/healthcheck/` → `coord/server/healthcheck/` (package clauses: `http` → `server`; healthcheck keeps its name)

- [ ] **Step 1: Moves + rewrites**

```bash
git mv coord/utils/ratelimiter coord/pkg/ratelimiter
rmdir coord/utils
git mv coord/infrastructure/server coord/server
mkdir -p coord/server
git mv coord/presentation/api/http/routes_http.go coord/server/http_routes.go
git mv coord/presentation/api/http/healthcheck coord/server/healthcheck
sed -i 's|^package http$|package server|' coord/server/http_routes.go
rewrite 'aioz-depin/coord/utils/ratelimiter' 'aioz-depin/coord/pkg/ratelimiter'
rewrite 'aioz-depin/coord/infrastructure/server' 'aioz-depin/coord/server'
rewrite 'aioz-depin/coord/presentation/api/http/healthcheck' 'aioz-depin/coord/server/healthcheck'
rewrite 'aioz-depin/coord/presentation/api/http' 'aioz-depin/coord/server'
```

⚠️ Check `coord/server/http_routes.go` symbol collisions with existing server package (`Server`, `Router`?) — build decides; rename `Router` → `HTTPRouter` if it collides, report. ⚠️ Check what imports the old http package first: `grep -rln 'presentation/api/http' --include='*.go' .` and fix qualifier `http.Router` → `server.Router` style references by hand (alias-aware: stdlib `net/http` shares the qualifier — verify each hit).

- [ ] **Step 2: Gate passes**
- [ ] **Step 3: Commit** — `refactor(coord): move ratelimiter to pkg, server + http plumbing to coord/server`

### Task 2: coord/db — central DB package; repositories move; UoW becomes internal engine

**Files:**
- Move: `coord/infrastructure/db/{db.go,database.go,embed.go}` → `coord/db/` (package name stays `db`)
- Move: `coord/infrastructure/repository/*.go` → `coord/db/` (package `repository` → `db`): `uow.go`, `worker.repository.go` → `worker_repo.go`, `placement.repository.go` → `placement_repo.go`, `file.go` → `file_repo.go`, `client.go` → `client_repo.go`, `wallet.go` → `wallet_repo.go`, `contract.go` → `contract_repo.go`, `storage.go` → `storage_repo.go`, `segment_piece_upload.go` → `segment_piece_upload_repo.go`, `reward.go` → `reward_repo.go`, `monkit.go` → keep name if no collision
- Modify: `coord/api.go`, `cmd/coord/main.go` (imports + `repository.NewUnitOfWork` → `db.NewUnitOfWork`)

- [ ] **Step 1: Move db then repositories**

```bash
git mv coord/infrastructure/db coord/db
rewrite 'aioz-depin/coord/infrastructure/db' 'aioz-depin/coord/db'
for pair in "uow.go uow.go" "worker.repository.go worker_repo.go" "placement.repository.go placement_repo.go" "file.go file_repo.go" "client.go client_repo.go" "wallet.go wallet_repo.go" "contract.go contract_repo.go" "storage.go storage_repo.go" "segment_piece_upload.go segment_piece_upload_repo.go" "reward.go reward_repo.go" "monkit.go db_monkit.go"; do set -- $pair; git mv "coord/infrastructure/repository/$1" "coord/db/$2"; done
sed -i 's|^package repository$|package db|' coord/db/uow.go coord/db/*_repo.go coord/db/db_monkit.go
rmdir coord/infrastructure/repository coord/infrastructure
rewrite 'aioz-depin/coord/infrastructure/repository' 'aioz-depin/coord/db'
rewrite 'repository\.NewUnitOfWork' 'db.NewUnitOfWork'
```

⚠️ Name collisions in merged `package db`: the repository files may define `var mon` colliding with db files (delete duplicate, keep one — worker Task 7 hit exactly this); `DB` interface vs repo types — build decides, report resolutions. ⚠️ `db.NewUnitOfWork` returns `repository.UnitOfWork` (the DOMAIN interface) — `coord/domain/repository` still exists until Task 9; the import stays for now. ⚠️ `cmd/coord/main.go` imports both old packages — fix; it also references `repository.X` symbols: check `grep -n 'repository\.' cmd/coord/main.go` and map each. ⚠️ The broad `repository\.` qualifier rewrite is NOT run repo-wide (it would hit `domain/repository` references in usecases, which must keep working until their subsystem task) — only the two targeted rewrites above.

- [ ] **Step 2: Gate passes**
- [ ] **Step 3: Commit** — `refactor(coord): create central coord/db package, fold GORM repositories into it`

### Task 3: wallet subsystem

**Files:**
- Create: `coord/wallet/` — `service.go` (← `application/usecase/wallet.go`), `store.go` (← `domain/repository/wallet.go` + `domain/repository/reward.go`), `types.go` (← `domain/entity/wallet.go`, `reward_plan.go`, `reward_statistic.go`, `reward_history.go`, `stripe.go`)
- Modify: `coord/db/wallet_repo.go`, `coord/db/reward_repo.go` (implement `wallet.Store` / `wallet.RewardStore`), `coord/db/uow.go` (accessor return types), `coord/api.go`

- [ ] **Step 1: Move files**

```bash
mkdir -p coord/wallet
git mv coord/application/usecase/wallet.go coord/wallet/service.go
git mv coord/domain/repository/wallet.go coord/wallet/store.go
git mv coord/domain/repository/reward.go coord/wallet/reward_store.go
git mv coord/domain/entity/wallet.go coord/wallet/types.go
git mv coord/domain/entity/reward_plan.go coord/wallet/reward_plan.go
git mv coord/domain/entity/reward_statistic.go coord/wallet/reward_statistic.go
git mv coord/domain/entity/reward_history.go coord/wallet/reward_history.go
git mv coord/domain/entity/stripe.go coord/wallet/stripe.go
sed -i 's|^package usecase$|package wallet|;s|^package repository$|package wallet|;s|^package entity$|package wallet|' coord/wallet/*.go
```

- [ ] **Step 2: Rename interface + rewrite users**

In `coord/wallet/store.go`: `WalletRepository` → `Store` (it has one method, `Create`). In `reward_store.go`: `RewardRepository` → `RewardStore`. In `service.go`: constructor becomes `NewWalletUsecase(log, password, walletDB Store)` (type name `WalletUsecase` kept — cosmetic renames deferred, same as worker). Drop self-qualifiers (`entity.Wallet` → `Wallet`, `repository.WalletRepository` → `Store`).

```bash
rewrite 'usecase\.WalletUsecase' 'wallet.WalletUsecase'
rewrite 'usecase\.NewWalletUsecase' 'wallet.NewWalletUsecase'
rewrite 'entity\.Wallet\b' 'wallet.Wallet'
rewrite 'entity\.RewardPlan' 'wallet.RewardPlan'
rewrite 'entity\.RewardStatistic' 'wallet.RewardStatistic'
rewrite 'entity\.RewardHistory' 'wallet.RewardHistory'
rewrite 'repository\.WalletRepository' 'wallet.Store'
rewrite 'repository\.RewardRepository' 'wallet.RewardStore'
```

Fixups: `coord/db/wallet_repo.go` + `reward_repo.go` implement the relocated interfaces (import `aioz-depin/coord/wallet`); `coord/db/uow.go` accessor `WalletRepository() wallet.Store`; `coord/domain/repository/uow.go` (domain interface) — update its `WalletRepository() wallet.Store` signature too (the domain UoW shrinks task by task and dies in Task 9). `coord/api.go` wiring. ⚠️ Cycle check: `grep -rn 'aioz-depin/coord/db' coord/wallet/` → must be empty.

- [ ] **Step 3: Gate passes**
- [ ] **Step 4: Commit** — `refactor(coord): create wallet subsystem`

### Task 4: client subsystem

**Files:**
- Create: `coord/client/` — `service.go` (← `usecase/client.go`), `store.go` (← `domain/repository/client.go`), `endpoint.go` (← `presentation/api/grpc/client.go`), `types.go` (← `domain/entity/client.go` + `clientplacement.go`)
- Modify: `coord/db/client_repo.go`, both uow.go files, `coord/api.go`

- [ ] **Step 1: Move + rename**

```bash
mkdir -p coord/client
git mv coord/application/usecase/client.go coord/client/service.go
git mv coord/domain/repository/client.go coord/client/store.go
git mv coord/presentation/api/grpc/client.go coord/client/endpoint.go
git mv coord/domain/entity/client.go coord/client/types.go
git mv coord/domain/entity/clientplacement.go coord/client/clientplacement.go
sed -i 's|^package usecase$|package client|;s|^package repository$|package client|;s|^package grpc$|package client|;s|^package entity$|package client|' coord/client/*.go
```

In `store.go`: `ClientRepository` → `Store`. In `service.go`: `NewClientUsecase(log, walletService *wallet.WalletUsecase, store Store)` — the UoW parameter is REPLACED by `Store` (ClientUsecase only consumes ClientRepository — verified in planning; if the build reveals more uow accessors used, add those as Store methods and report).

```bash
rewrite 'usecase\.ClientUsecase' 'client.ClientUsecase'
rewrite 'usecase\.NewClientUsecase' 'client.NewClientUsecase'
rewrite 'grpc2\.ClientEndpoint' 'client.Endpoint'
rewrite 'grpc2\.NewClientEndpoint' 'client.NewEndpoint'
rewrite 'entity\.Client\b' 'client.Client'
rewrite 'repository\.ClientRepository' 'client.Store'
```

(`grpc2` is the alias api.go uses for the grpc endpoint package — verify with `grep -n 'grpc2\|presentation/api/grpc' coord/api.go` and adapt the rewrites to the actual alias.) Rename declarations `ClientEndpoint` → `Endpoint`, `NewClientEndpoint` → `NewEndpoint` inside endpoint.go (endpoint stutter rule, same as worker contact). db/uow fixups as in Task 3. ⚠️ `entity.Client` is referenced by `entity.Contract`? Check `grep -rn 'Client' coord/domain/entity/contract.go` — if Contract embeds/references Client, Contract (moves in Task 5) will import `coord/client` — fine (storage→client direction), but verify no cycle (client must not import storage).

- [ ] **Step 2: Gate passes** (including `go test ./coord/...` — file_test.go still compiles via remaining usecase package)
- [ ] **Step 3: Commit** — `refactor(coord): create client subsystem`

### Task 5: storage subsystem (contracts)

**Files:**
- Create: `coord/storage/` — `service.go` (← `usecase/storage.go`), `store.go` (← `domain/repository/storage.go` + `contract.go`), `endpoint.go` (← `presentation/api/grpc/storage.go`), `types.go` (← `domain/entity/storage.go` + `contract.go`)
- Modify: `coord/db/{storage_repo,contract_repo}.go`, uow files, `coord/api.go`

- [ ] **Step 1: Move + rename**

```bash
mkdir -p coord/storage
git mv coord/application/usecase/storage.go coord/storage/service.go
git mv coord/domain/repository/storage.go coord/storage/store.go
git mv coord/domain/repository/contract.go coord/storage/contract_store.go
git mv coord/presentation/api/grpc/storage.go coord/storage/endpoint.go
git mv coord/domain/entity/storage.go coord/storage/types.go
git mv coord/domain/entity/contract.go coord/storage/contract.go
sed -i 's|^package usecase$|package storage|;s|^package repository$|package storage|;s|^package grpc$|package storage|;s|^package entity$|package storage|' coord/storage/*.go
```

Interfaces: `StorageRepository` → `Store`; `ContractRepository` → `ContractStore`. `StorageUsecase` already takes individual repos (`NewStorageUsecase(log, storageDB Store, contractDB ContractStore, clientDB client.Store)`) — minimal change, the client repo param becomes `client.Store`.

```bash
rewrite 'usecase\.StorageUsecase' 'storage.StorageUsecase'
rewrite 'usecase\.NewStorageUsecase' 'storage.NewStorageUsecase'
rewrite 'entity\.StorageConfig' 'storage.StorageConfig'
rewrite 'entity\.GlobalStorageConfigID' 'storage.GlobalStorageConfigID'
rewrite 'entity\.Contract\b' 'storage.Contract'
rewrite 'repository\.StorageRepository' 'storage.Store'
rewrite 'repository\.ContractRepository' 'storage.ContractStore'
```

Endpoint declarations `StorageEndpoint` → `Endpoint`, `NewStorageEndpoint` → `NewEndpoint` (adapt the api.go alias rewrites as in Task 4). ⚠️ FileUsecase (still in `usecase/`) consumes Contract/Storage repos via uow — its `uow.ContractRepository()` etc. calls now return `storage.ContractStore`; the usecase package needs the `coord/storage` import (temporary, dies in Task 8). Report this temp edge.

- [ ] **Step 2: Gate passes**
- [ ] **Step 3: Commit** — `refactor(coord): create storage subsystem`

### Task 6: contact subsystem (owns Worker records)

**Files:**
- Create: `coord/contact/` — `service.go` (← `usecase/contact.go`, incl. `ContactConfig` → `Config`), `store.go` (← `domain/repository/worker.go`, `WorkerRepository` → `Store`), `endpoint.go` (← `presentation/api/grpc/contact.go`), `worker.go` (← `domain/entity/worker.go`), `uptime.go` (← `domain/services/uptime_calculator.go`)
- Modify: `coord/db/worker_repo.go`, uow files, `coord/api.go`, `coord/config.go` (Contact config type)

- [ ] **Step 1: Move + rename**

```bash
mkdir -p coord/contact
git mv coord/application/usecase/contact.go coord/contact/service.go
git mv coord/domain/repository/worker.go coord/contact/store.go
git mv coord/presentation/api/grpc/contact.go coord/contact/endpoint.go
git mv coord/domain/entity/worker.go coord/contact/worker.go
git mv coord/domain/services/uptime_calculator.go coord/contact/uptime.go
sed -i 's|^package usecase$|package contact|;s|^package repository$|package contact|;s|^package grpc$|package contact|;s|^package entity$|package contact|;s|^package services$|package contact|' coord/contact/*.go
rmdir coord/domain/services
```

Renames inside the package: `ContactUsecase` → `Service` is DEFERRED (keep `ContactUsecase`); `ContactConfig` → `Config`; `WorkerRepository` → `Store`; `ContactEndpoint` → `Endpoint`, `NewContactEndpoint` → `NewEndpoint`.

```bash
rewrite 'usecase\.ContactUsecase' 'contact.ContactUsecase'
rewrite 'usecase\.NewContactUsecase' 'contact.NewContactUsecase'
rewrite 'usecase\.ContactConfig' 'contact.Config'
rewrite 'entity\.Worker\b' 'contact.Worker'
rewrite 'repository\.WorkerRepository' 'contact.Store'
```

⚠️ `coord/config.go` has `Contact usecase.ContactConfig` — becomes `contact.Config`; this changes NO config keys (field name `Contact` unchanged — verify with `grep -n 'Contact' coord/config.go` and confirm no cfgstruct key derives from the TYPE name). ⚠️ `ContactUsecase` takes `uow` — planning says it consumes only WorkerRepository: constructor becomes `NewContactUsecase(log, config Config, dialer dial.Dialer, store Store)`; if the build reveals more uow usage, extend Store and report. ⚠️ WorkerSelection + File usecases (still in `usecase/`) call `uow.WorkerRepository()` returning `contact.Store` — temp import of `coord/contact` in usecase package (dies Task 7/8). ⚠️ cmd/coord/main.go references `entity.Worker`? (it imported entity — check `grep -n 'entity\.' cmd/coord/main.go`, map each to its new home, likely contact.Worker for seeding.)

- [ ] **Step 2: Gate passes**
- [ ] **Step 3: Commit** — `refactor(coord): create contact subsystem owning worker records`

### Task 7: placement subsystem (incl. worker selection)

**Files:**
- Create: `coord/placement/` — `service.go` (← `usecase/placement.go`), `selection.go` (← `usecase/worker_selection.go`), `store.go` (← `domain/repository/placement.go`), `endpoint.go` (← `presentation/api/grpc/placement.go`), entities: `placement.go`, `placement_constraint.go`, `strategies.go`, `worker_filters.go`, `filter_wrapper.go`, `reservoir.go`, `audit.go` (all ← `domain/entity/`)
- Modify: `coord/db/placement_repo.go`, uow files, `coord/api.go`, `usecase/usecase.go` (`WorkerConfig` — moves here if it's selection config; verify its consumer first)

- [ ] **Step 1: Move + rename**

```bash
mkdir -p coord/placement
git mv coord/application/usecase/placement.go coord/placement/service.go
git mv coord/application/usecase/worker_selection.go coord/placement/selection.go
git mv coord/domain/repository/placement.go coord/placement/store.go
git mv coord/presentation/api/grpc/placement.go coord/placement/endpoint.go
for f in placement.go placement_constraint.go strategies.go worker_filters.go filter_wrapper.go reservoir.go audit.go; do git mv "coord/domain/entity/$f" "coord/placement/$f"; done
sed -i 's|^package usecase$|package placement|;s|^package repository$|package placement|;s|^package grpc$|package placement|;s|^package entity$|package placement|' coord/placement/*.go
```

`PlacementRepository` → `Store`; `PlacementEndpoint`/`NewPlacementEndpoint` → `Endpoint`/`NewEndpoint`. Constructors: `NewPlacementUsecase(store Store)`; `NewWorkerSelectionUseCase(log, store Store, workers contact.Store, onlineWindow time.Duration)` — the uow parameter splits into the two stores it actually uses (verified in planning: PlacementRepository + WorkerRepository).

```bash
rewrite 'usecase\.PlacementUsecase' 'placement.PlacementUsecase'
rewrite 'usecase\.NewPlacementUsecase' 'placement.NewPlacementUsecase'
rewrite 'usecase\.WorkerSelectionUseCase' 'placement.WorkerSelectionUseCase'
rewrite 'usecase\.NewWorkerSelectionUseCase' 'placement.NewWorkerSelectionUseCase'
rewrite 'repository\.PlacementRepository' 'placement.Store'
for t in Placement PlacementConstraint SelectionRequest SelectionResult ECParameters RandomSelectionStrategy CountryFilter WorkerFilterWrapper Reservoir AuditSegment AuditOutcome AuditReport AuditConfig; do rewrite "entity\\.$t\\b" "placement.$t"; done
```

⚠️ Check each `entity.X` rewrite's hit list FIRST (`grep -rn 'entity\.Placement\b' --include='*.go' .`) — `coord/db/placement_repo.go` and remaining usecase files are expected; anything else, inspect. ⚠️ `entity.Placement` may reference `entity.Worker` (now `contact.Worker`) — placement importing contact is the intended direction. Cycle check: `grep -rn 'aioz-depin/coord/placement' coord/contact/` → empty. ⚠️ `usecase/usecase.go` (`WorkerConfig`/AuditTime): `grep -rn 'WorkerConfig' --include='*.go' .` — if consumed by selection/audit code, move into `coord/placement/service.go`; if unused, move anyway (zero-consumer stub) and report.

- [ ] **Step 2: Gate passes**
- [ ] **Step 3: Commit** — `refactor(coord): create placement subsystem with worker selection`

### Task 8: file subsystem (metainfo) — dissolves the UoW from usecases

**Files:**
- Create: `coord/file/` — `service.go` (← `usecase/file.go`), `service_test.go` (← `usecase/file_test.go`), `store.go` (NEW interface, see below), `endpoint.go` (← `presentation/api/grpc/file.go`), entities: `file.go`, `segment.go`, `segment_piece_upload.go`, `object.go`, `order.go`, `redundancy_scheme.go`, `transferlog.go`, `task.go` (← `domain/entity/`)
- Delete: `coord/domain/repository/{file.go,segment_piece_upload.go,transferlog.go}` (absorbed into file.Store)
- Modify: `coord/db/{file_repo,segment_piece_upload_repo}.go`, `coord/db/uow.go`, `coord/api.go`

- [ ] **Step 1: Move entities + endpoint + service**

```bash
mkdir -p coord/file
git mv coord/application/usecase/file.go coord/file/service.go
git mv coord/application/usecase/file_test.go coord/file/service_test.go
git mv coord/presentation/api/grpc/file.go coord/file/endpoint.go
for f in file.go segment.go segment_piece_upload.go object.go order.go redundancy_scheme.go transferlog.go task.go; do git mv "coord/domain/entity/$f" "coord/file/$f"; done
sed -i 's|^package usecase$|package file|;s|^package usecase_test$|package file_test|;s|^package grpc$|package file|;s|^package entity$|package file|' coord/file/*.go
```

⚠️ Collision: entity `file.go` content merging with service — the entity file becomes `coord/file/file.go` while service is `service.go`; no filename collision, but TYPE collisions inside package `file` are possible (e.g. service-local helper types named like entity types) — build decides, report.

- [ ] **Step 2: Define file.Store** in `coord/file/store.go` (NEW file) by composing the absorbed interfaces — copy method sets VERBATIM from `domain/repository/file.go` + `segment_piece_upload.go` + `transferlog.go`:

```go
package file

import (
	"context"

	"aioz-depin/internal/vo"
)

// Store is the file subsystem's persistence interface, implemented by coord/db.
// It absorbs the former FileRepository, SegmentPieceUploadRepository and
// TransferLogRepository, plus the transaction runner the file service needs.
type Store interface {
	// former FileRepository — methods copied verbatim
	Create(ctx context.Context, file *File) error
	CreateSegment(ctx context.Context, segment *Segment) error
	GetByID(ctx context.Context, id vo.UUID) (*File, error)
	GetSegmentByFileIDAndNumber(ctx context.Context, fileID vo.UUID, number int32) (*Segment, error)
	ListSegmentsByFile(ctx context.Context, fileID vo.UUID) ([]*Segment, error)
	Update(ctx context.Context, file *File) error

	// former SegmentPieceUploadRepository — methods copied verbatim
	CreateBatch(ctx context.Context, rows []*SegmentPieceUpload) error
	GetBySerial(ctx context.Context, serial []byte) (*SegmentPieceUpload, error)
	GetBySegmentAndPiece(ctx context.Context, segmentID vo.UUID, pieceNum int32) (*SegmentPieceUpload, error)
	UpdatePieceUpload(ctx context.Context, row *SegmentPieceUpload) error // was Update; renamed to avoid clash with file Update
	CountConfirmedBySegment(ctx context.Context, segmentID vo.UUID) (int64, error)

	// former TransferLogRepository
	CreateTransferLog(ctx context.Context, log *TransferLog) error // was Create; renamed to avoid clash

	// WithTx runs fn inside a DB transaction; the Store passed to fn is tx-scoped.
	WithTx(ctx context.Context, fn func(Store) error) error
}
```

⚠️ The exact signatures MUST be taken from the actual repo files at execution time — the above reflects planning data; adjust verbatim-copy + the two documented clash renames (`UpdatePieceUpload`, `CreateTransferLog`). `coord/db` implements `Store` with a new `fileStore` struct that wraps the existing `fileRepository` + `segmentPieceUploadRepository` (+transferlog if an impl exists; if no impl exists for transferlog, implement the one method with the same GORM pattern as its siblings and report). `WithTx` reuses the existing UoW transaction machinery.

- [ ] **Step 3: Rewire the service.** `NewFileUsecase(log, workerSelectionUsecase *placement.WorkerSelectionUseCase, signer signing.Signer, store Store, contracts storage.ContractStore, storageCfg storage.Store, workers contact.Store)` — replace every `f.uow.XRepository()` call with the corresponding injected store/interface; `uow.WithTx(func(uow) ...)` becomes `store.WithTx(func(s Store) ...)`. This is the ONE task with real signature surgery: keep each method body's logic IDENTICAL — only the data-access receiver changes. Report every call-site change.

```bash
rewrite 'usecase\.FileUsecase' 'file.FileUsecase'
rewrite 'usecase\.NewFileUsecase' 'file.NewFileUsecase'
rewrite 'usecase\.Metadata' 'file.Metadata'
rewrite 'usecase\.PresignedInfo' 'file.PresignedInfo'
for t in File Segment SegmentPieceUpload Object ObjectStatus OrderLimit RedundancyScheme EncryptionParameters TransferLog Task; do rewrite "entity\\.$t\\b" "file.$t"; done
git rm coord/domain/repository/file.go coord/domain/repository/segment_piece_upload.go coord/domain/repository/transferlog.go
```

⚠️ `entity.OrderLimit` check first: planning says both `segment.go` and `order.go` reference OrderLimit — confirm only ONE declaration exists (`grep -rn 'type OrderLimit' coord/`). ⚠️ Endpoint declarations `FileEndpoint`/`NewFileEndpoint` → `Endpoint`/`NewEndpoint`. ⚠️ `coord/api_upload_integration_test.go` references usecase/entity types — update its imports/qualifiers (it tests through the peer; keep assertions identical).

- [ ] **Step 4: Gate passes** (file_test.go + integration test compile is the key signal)
- [ ] **Step 5: Commit** — `refactor(coord): create file subsystem, dissolve UnitOfWork from services`

### Task 9: peer.go + final dissolution

**Files:**
- Rename: `coord/api.go` → `coord/peer.go`; type `API` → `Peer`, `NewAPI` → `NewPeer`
- Move: `coord/domain/vo/` → `coord/vo/` (package unchanged)
- Delete: `coord/domain/repository/uow.go` (domain UoW interface — by now references only relocated types), then `coord/domain/`, `coord/application/`, `coord/presentation/` trees (must be empty of .go files)
- Modify: `coord/db/uow.go` — `NewUnitOfWork` stops returning the domain interface; `coord/db` exposes typed accessors (`func (u *UnitOfWork) Workers() contact.Store` etc.) or the Peer constructs stores directly; pick the minimal change that compiles and report
- Modify: `cmd/coord/main.go` (`coord.NewAPI` → `coord.NewPeer`)

- [ ] **Step 1: vo move + DDD teardown**

```bash
git mv coord/domain/vo coord/vo
rewrite 'aioz-depin/coord/domain/vo' 'aioz-depin/coord/vo'
git rm coord/domain/repository/uow.go
find coord/domain coord/application coord/presentation -name '*.go' | (! grep .)   # MUST be empty
git rm -r coord/domain coord/application coord/presentation 2>/dev/null; true
```

- [ ] **Step 2: API → Peer rename** (worker Task 16 recipe: `git mv coord/api.go coord/peer.go`; `rewrite 'coord\.NewAPI' 'coord.NewPeer'`; `rewrite 'coord\.API\b' 'coord.Peer'`; in-file declaration renames; construction order byte-identical; verify with the sed-normalize diff trick from worker Task 16 review)
- [ ] **Step 3: Gate passes**; layer audit:

```bash
grep -rn 'aioz-depin/coord/\(contact\|storage\|file\|placement\|client\|wallet\|db\|server\)"' coord/pkg/   # empty
grep -rn 'coord/domain\|coord/application\|coord/presentation\|coord/utils\|coord/infrastructure' --include='*.go' .   # empty
grep -rln 'aioz-depin/coord/db' coord/contact coord/storage coord/file coord/placement coord/client coord/wallet | grep -v _test   # empty (no subsystem imports db)
```

- [ ] **Step 4: Commit** — `refactor(coord): hand-wired peer, delete DDD trees`

**Pre-existing bugs surfaced during execution (for the Task 10 report, NOT mechanical-commit fixes):**
- `coord/placement/selection.go:51` — empty `if placement.SelectionStrategy == nil {}` dead statement (verbatim from base)
- `coord/placement/filter_wrapper.go:84` vs `:151` — CountryFilter serializes params key `"country"` but deserializes `"countries"`: persisted country filters fail to reload (deprecated filter path; verbatim from base)
- `coord/wallet/stripe.go` — ~40 lines commented-out erasure-coding dead code (from base)
- Reward stub files in `coord/wallet/` are unwired (interface + panic-stub impl only)
- Task 8 must unify the duplicated empty `SegmentPosition` (placement/audit.go vs entity/segment.go) when segment moves

### Task 10: Final audit

- [ ] **Step 1: Full gate** + `go test ./coord/... -count=1` (NO failure allowance — coord baseline is green)
- [ ] **Step 2: Integration test** — `COORD_INTEGRATION=1 go test ./coord/ -run TestUploadHappyPath_OneWorker -count=1` (check the actual env-var name in the test file first); if it needs live services, build-compile it only (`go vet ./coord/`) and report
- [ ] **Step 3: Binary smoke** — `go build -o /tmp/coord-smoke ./cmd/coord && /tmp/coord-smoke --help | head -20`
- [ ] **Step 4: Layer audit greps** (Task 9 Step 3 set) repeated; commit any fixes; report final structure + commit list
