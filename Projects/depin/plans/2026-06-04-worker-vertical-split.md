---
type: plan
project: depin
tags: [depin, worker, vertical-split, refactor]
created: 2026-06-04
---

# Worker Vertical Split Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **⚠️ COMMIT POLICY:** The user requires explicit approval for every commit. At each "Commit" step, STOP and ask the user before running `git commit`. Never push.

**Goal:** Restructure `worker/` from DDD layers (`domain/application/infrastructure/presentation`) into vertical subsystems (`contact/`, `piecestore/`, `orders/`, `trust/`, `retain/`, `monitor/`, `version/`), each holding Endpoint/Service/Store/Chore, wired by a hand-written Peer; engines move to `worker/pkg/`.

**Architecture:** Per `docs/superpowers/specs/2026-06-04-vertical-split-restructure-design.md`. Migration is subsystem-by-subsystem; the build is green and the worker test suite passes after every task. Moves are `git mv` + repo-wide import rewrites; type names are kept unchanged when files merge into a subsystem package (only the package qualifier changes, e.g. `usecase.TrustService` → `trust.TrustService`) — cosmetic renames are out of scope except where the spec names one (Contract→Contact).

**Tech Stack:** Go 1.x, module `aioz-depin`, monorepo. No new dependencies. `google/wire` is removed from `worker/`.

**Spec amendments discovered during planning** (already reflected below; spec updated alongside this plan):
1. `worker/` exists only on `feat/worker-contact` (190 files vs `main`) — the restructure branch is based on `feat/worker-contact`, not `main`.
2. Baseline build is broken (`worker/infrastructure/trust/http_source.go:15` unused import) — fixed in Task 0.
3. A separate `worker/pieces/` package keeps the piece store (`piece.Store`) out of `worker/piecestore/` to avoid an import cycle: `piecestore` (HashStoreBackend, Service) imports `retain` managers, and `retain` imports `piece.Store`.
4. `coordstore` and `p2pc` remain separate packages under `worker/pkg/` (no merged `coordclient` package — merging adds churn without benefit).
5. `worker.Core` (p2p registration path, built by `worker/di` wire) is hand-wired by copying the already-generated explicit code from `worker/di/wire_gen.go` into the `worker` package.

---

## Conventions used by every task

**Import rewrite helper** (run from repo root; define once per shell):

```bash
rewrite() { grep -rl "$1" --include='*.go' . | xargs -r sed -i "s|$1|$2|g"; }
```

**Gate** (run after every task; "Gate passes" below means exactly this):

```bash
go build ./... && go vet ./worker/... ./cmd/worker/... ./cmd/keytool/... && \
go test ./worker/... ./cmd/worker/... ./cmd/keytool/...
```

If a test fails that also fails before your change (record baseline in Task 0), it is pre-existing — note it, don't fix it here.

**Move pattern** for "move package, same package name": `git mv` the directory, then `rewrite` the import path. No file content changes needed.

**Merge pattern** for "move file into another package": `git mv` the file, change its `package` clause, then `rewrite` usages of the old qualifier (e.g. `usecase.TrustService` → `trust.TrustService`) in all importers, and remove now-unneeded imports (the build errors will list them).

---

### Task 0: Preflight — baseline fix and test inventory

**Files:**
- Modify: `worker/infrastructure/trust/http_source.go:15` (remove unused `"go.uber.org/zap"` import)

- [ ] **Step 1: Confirm starting point**

```bash
git status --porcelain   # expect: only uplink-sdk/* modifications + untracked docs/gen-piece-key
git branch --show-current  # expect: feat/worker-contact
```

If uplink-sdk changes are uncommitted, ask the user to commit/stash them first — this plan must not mix with them.

- [ ] **Step 2: Reproduce the baseline build break**

Run: `go build ./worker/...`
Expected: FAIL with `worker/infrastructure/trust/http_source.go:15:2: "go.uber.org/zap" imported and not used`

- [ ] **Step 3: Remove the unused import**

In `worker/infrastructure/trust/http_source.go`, delete the line `"go.uber.org/zap"` from the import block.

- [ ] **Step 4: Record the test baseline**

```bash
go build ./... && go test ./worker/... ./cmd/worker/... ./cmd/keytool/... 2>&1 | tee /tmp/worker-test-baseline.txt
```

Expected: build PASSES. Save which tests (if any) fail — that list is the pre-existing-failure allowance for all later gates.

- [ ] **Step 5: Commit (ASK USER FIRST)**

```bash
git add worker/infrastructure/trust/http_source.go
git commit -m "fix: remove unused zap import breaking worker build"
```

### Task 1: Create the restructure branch

- [ ] **Step 1: Branch off feat/worker-contact**

```bash
git checkout -b refactor/worker-vertical-split
```

Expected: new branch at the same commit as `feat/worker-contact`.

### Task 2: Move hashstore and blobstore to worker/pkg/

Pure moves — package names unchanged, only import paths change.

**Files:**
- Move: `worker/infrastructure/hashstore/` → `worker/pkg/hashstore/` (incl. `platform/`)
- Move: `worker/infrastructure/blobstore/` → `worker/pkg/blobstore/` (incl. `dir/`, `filestore/`)

- [ ] **Step 1: Move directories**

```bash
git mv worker/infrastructure/hashstore worker/pkg/hashstore
git mv worker/infrastructure/blobstore worker/pkg/blobstore
```

- [ ] **Step 2: Rewrite import paths repo-wide**

```bash
rewrite 'aioz-depin/worker/infrastructure/hashstore' 'aioz-depin/worker/pkg/hashstore'
rewrite 'aioz-depin/worker/infrastructure/blobstore' 'aioz-depin/worker/pkg/blobstore'
```

Known importers to verify compile: `worker/infrastructure/piecestore/backend.go`, `worker/infrastructure/piece/{store,cache,filewalker}.go`, `worker/core.go` (filestore config).

- [ ] **Step 3: Gate passes**
- [ ] **Step 4: Commit (ASK USER FIRST)** — `refactor(worker): move hashstore and blobstore engines to worker/pkg`

### Task 3: Move du, space, systeminfo, filemanager to worker/pkg/

**Files:**
- Move: `worker/infrastructure/du/` → `worker/pkg/du/`
- Move: `worker/infrastructure/space/` → `worker/pkg/space/`
- Move: `worker/infrastructure/systeminfo/` → `worker/pkg/sysinfo/` (package renamed `systeminfo` → `sysinfo`)
- Move: `worker/infrastructure/filemanager/` → `worker/pkg/filemanager/`

Note: the interface files `worker/infrastructure/space.go` (`infrastructure.SpaceReport`) and `worker/infrastructure/systeminfo.go` (`infrastructure.SystemInfoReporter`) do NOT move yet — they go to the monitor subsystem in Task 13.

- [ ] **Step 1: Move directories**

```bash
git mv worker/infrastructure/du worker/pkg/du
git mv worker/infrastructure/space worker/pkg/space
git mv worker/infrastructure/systeminfo worker/pkg/sysinfo
git mv worker/infrastructure/filemanager worker/pkg/filemanager
```

- [ ] **Step 2: Rename the sysinfo package clause and qualifier**

```bash
sed -i 's|^package systeminfo$|package sysinfo|' worker/pkg/sysinfo/*.go
rewrite 'aioz-depin/worker/infrastructure/systeminfo' 'aioz-depin/worker/pkg/sysinfo'
rewrite 'systeminfo\.' 'sysinfo.'
```

⚠️ The last rewrite is qualifier-wide: afterwards run `grep -rn 'sysinfo\.' --include='*.go' . | grep -v 'pkg/sysinfo'` and eyeball that every hit really refers to this package (the old name is unique enough that collisions are unlikely, but verify).

- [ ] **Step 3: Rewrite the remaining import paths**

```bash
rewrite 'aioz-depin/worker/infrastructure/du' 'aioz-depin/worker/pkg/du'
rewrite 'aioz-depin/worker/infrastructure/space' 'aioz-depin/worker/pkg/space'
rewrite 'aioz-depin/worker/infrastructure/filemanager' 'aioz-depin/worker/pkg/filemanager'
```

- [ ] **Step 4: Gate passes**
- [ ] **Step 5: Commit (ASK USER FIRST)** — `refactor(worker): move du, space, sysinfo, filemanager to worker/pkg`

### Task 4: Move cron, signaturecheck, speedestimation to worker/pkg/; absorb CronRunner

**Files:**
- Move: `worker/infrastructure/cron/` → `worker/pkg/cron/`
- Move: `worker/utils/signaturecheck/` → `worker/pkg/signaturecheck/`
- Move: `worker/utils/speedestimation/` → `worker/pkg/speedestimation/`
- Modify: `worker/pkg/cron/cron.go` (gains the `CronRunner` interface from `worker/application/usecase/ports/cron.go`)
- Delete: `worker/application/usecase/ports/cron.go`

- [ ] **Step 1: Move directories and rewrite paths**

```bash
git mv worker/infrastructure/cron worker/pkg/cron
git mv worker/utils/signaturecheck worker/pkg/signaturecheck
git mv worker/utils/speedestimation worker/pkg/speedestimation
rewrite 'aioz-depin/worker/infrastructure/cron' 'aioz-depin/worker/pkg/cron'
rewrite 'aioz-depin/worker/utils/signaturecheck' 'aioz-depin/worker/pkg/signaturecheck'
rewrite 'aioz-depin/worker/utils/speedestimation' 'aioz-depin/worker/pkg/speedestimation'
```

- [ ] **Step 2: Move the CronRunner interface into the cron package**

Copy the `CronRunner` interface declaration from `worker/application/usecase/ports/cron.go` into `worker/pkg/cron/cron.go` verbatim (it is `StartBackground`, `Timer`, `Name`), then delete `worker/application/usecase/ports/cron.go`. Rewrite users:

```bash
rewrite 'ports\.CronRunner' 'cron.CronRunner'
```

Users to fix up (add `aioz-depin/worker/pkg/cron` import where missing): `worker/di/provider.go` (`ProvideCronRunners`), `worker/application/usecase/speedtest.go`, `worker/pkg/cron/cron.go` itself (drop its old `ports` import — this also removes a pkg→subsystem layer violation).

- [ ] **Step 3: Gate passes**
- [ ] **Step 4: Commit (ASK USER FIRST)** — `refactor(worker): move cron and util packages to worker/pkg, absorb CronRunner`

### Task 5: Move coordstore and p2pc to worker/pkg/

**Files:**
- Move: `worker/infrastructure/coordstore/` → `worker/pkg/coordstore/`
- Move: `worker/infrastructure/p2pc/` → `worker/pkg/p2pc/`

- [ ] **Step 1: Move and rewrite**

```bash
git mv worker/infrastructure/coordstore worker/pkg/coordstore
git mv worker/infrastructure/p2pc worker/pkg/p2pc
rewrite 'aioz-depin/worker/infrastructure/coordstore' 'aioz-depin/worker/pkg/coordstore'
rewrite 'aioz-depin/worker/infrastructure/p2pc' 'aioz-depin/worker/pkg/p2pc'
```

Known importers: `worker/infrastructure/retain/{bloom_manager,restore_manager}.go`, `worker/application/usecase/{piecestore,worker_info}.go`, `worker/di/provider.go`, `worker/infrastructure/reporter.go`.

- [ ] **Step 2: Gate passes**
- [ ] **Step 3: Commit (ASK USER FIRST)** — `refactor(worker): move coordstore and p2pc clients to worker/pkg`

### Task 6: Build worker/pkg/keytool from usecase/key + infrastructure/key; fix layer violations

This consolidates key handling and fixes two violations: `pkg/key` (root) importing `worker/application/usecase/ports`, and `cmd/keytool` reaching into worker DDD internals.

**Files:**
- Move: `worker/application/usecase/key/{key.go,key_test.go,tls.go,tls_test.go}` → `worker/pkg/keytool/` (package `key` → `keytool`)
- Move: `worker/infrastructure/key/cosmos_keytool.go` → `worker/pkg/keytool/cosmos_keytool.go`
- Move: `worker/domain/valueobjects/keytool.go` → `worker/pkg/keytool/dto.go`
- Move: `KeyProvider` interface from `worker/application/usecase/ports/key.go` → `worker/pkg/keytool/`
- Modify: `pkg/key/key.go`, `cmd/keytool/command.go`, `cmd/worker/helpers.go`, `worker/di/provider.go`, `worker/application/usecase/autoupdate.go`

- [ ] **Step 1: Move files into the new package**

```bash
mkdir -p worker/pkg/keytool
git mv worker/application/usecase/key/key.go worker/pkg/keytool/keytool.go
git mv worker/application/usecase/key/key_test.go worker/pkg/keytool/keytool_test.go
git mv worker/application/usecase/key/tls.go worker/pkg/keytool/tls.go
git mv worker/application/usecase/key/tls_test.go worker/pkg/keytool/tls_test.go
git mv worker/infrastructure/key/cosmos_keytool.go worker/pkg/keytool/cosmos_keytool.go
git mv worker/domain/valueobjects/keytool.go worker/pkg/keytool/dto.go
sed -i 's|^package key$|package keytool|;s|^package valueobjects$|package keytool|' worker/pkg/keytool/*.go
rmdir worker/application/usecase/key worker/infrastructure/key
```

- [ ] **Step 2: Move the KeyProvider interface**

Cut the `KeyProvider` interface from `worker/application/usecase/ports/key.go` and paste it verbatim into `worker/pkg/keytool/keytool.go` (above `KeyTool`). Delete `worker/application/usecase/ports/key.go`.

- [ ] **Step 3: Rewrite importers**

```bash
rewrite 'aioz-depin/worker/application/usecase/key' 'aioz-depin/worker/pkg/keytool'
rewrite 'aioz-depin/worker/infrastructure/key' 'aioz-depin/worker/pkg/keytool'
rewrite 'ports\.KeyProvider' 'keytool.KeyProvider'
rewrite 'key\.NewKeyTool' 'keytool.NewKeyTool'
rewrite 'key\.NewCosmosKeyTool' 'keytool.NewCosmosKeyTool'
rewrite 'valueobjects\.KeyToolRequest' 'keytool.KeyToolRequest'
rewrite 'valueobjects\.KeyToolResponse' 'keytool.KeyToolResponse'
```

Then fix the affected files by hand (imports/aliases): `pkg/key/key.go` (now imports `aioz-depin/worker/pkg/keytool` instead of `.../usecase/ports`), `cmd/keytool/command.go`, `cmd/worker/helpers.go`, `worker/di/provider.go` (drop the `keytool "aioz-depin/worker/application/usecase/key"` alias), `worker/application/usecase/autoupdate.go`. Duplicate-symbol risk: `keytool.go` and `cosmos_keytool.go` are now one package — if both define a symbol with the same name the build will say so; resolve by prefixing the cosmos one (`CosmosX`).

- [ ] **Step 4: Gate passes**
- [ ] **Step 5: Commit (ASK USER FIRST)** — `refactor(worker): consolidate key handling into worker/pkg/keytool`

### Task 7: Create worker/db and worker/server

**Files:**
- Move: `worker/infrastructure/db/` → `worker/db/` (incl. `dbtest/`)
- Move: `worker/infrastructure/repository/{coord.go,piece.go,trust.go,mon.go}` → `worker/db/` (package `repository` → `db`)
- Move: `worker/infrastructure/server/` → `worker/server/`

- [ ] **Step 1: Move db and server**

```bash
git mv worker/infrastructure/db worker/db
git mv worker/infrastructure/server worker/server
rewrite 'aioz-depin/worker/infrastructure/db' 'aioz-depin/worker/db'
rewrite 'aioz-depin/worker/infrastructure/server' 'aioz-depin/worker/server'
```

- [ ] **Step 2: Fold the GORM repositories into worker/db**

```bash
git mv worker/infrastructure/repository/coord.go worker/db/coord.go
git mv worker/infrastructure/repository/piece.go worker/db/piece_repos.go
git mv worker/infrastructure/repository/mon.go worker/db/mon.go
git rm worker/infrastructure/repository/trust.go   # empty file, verified during planning
sed -i 's|^package repository$|package db|' worker/db/coord.go worker/db/piece_repos.go worker/db/mon.go
rewrite 'aioz-depin/worker/infrastructure/repository' 'aioz-depin/worker/db'
rewrite 'repository\.New' 'db.New'
rmdir worker/infrastructure/repository
```

Collision check: `worker/db` already has `package db` files (`database.go`, `db.go`, `embed.go`, `utils.go`); if `mon.go` collides with an existing `mon` var, rename the moved one's var (build error will name it).

- [ ] **Step 3: Gate passes**
- [ ] **Step 4: Commit (ASK USER FIRST)** — `refactor(worker): move db and server to component root, fold repositories into db`

### Task 8: trust subsystem

**Files:**
- Move: `worker/infrastructure/trust/` → `worker/trust/` (all files, package name unchanged)
- Move: `worker/application/usecase/trust.go` → `worker/trust/service.go`; `trust_test.go` → `worker/trust/service_test.go` (package `usecase` → `trust`)
- Move: `worker/domain/entity/peer.go` (CoordDB, CoordStatus, Peer, ExitProgress) → `worker/trust/coorddb.go`

No name collisions: verified during planning (`TrustService`, `TrustConfig`, `IdentityResolver`, `coordInfoCache` don't exist in `infrastructure/trust`).

- [ ] **Step 1: Move the infrastructure package**

```bash
git mv worker/infrastructure/trust worker/trust
rewrite 'aioz-depin/worker/infrastructure/trust' 'aioz-depin/worker/trust'
```

- [ ] **Step 2: Merge the usecase service in**

```bash
git mv worker/application/usecase/trust.go worker/trust/service.go
git mv worker/application/usecase/trust_test.go worker/trust/service_test.go
sed -i 's|^package usecase$|package trust|' worker/trust/service.go
sed -i 's|^package usecase_test$|package trust_test|;s|^package usecase$|package trust|' worker/trust/service_test.go
```

In `worker/trust/service.go`, delete the now-self import of `aioz-depin/worker/trust` and drop the `trust.` qualifier on its symbols (e.g. `trust.TrustedCoordSource` → `TrustedCoordSource`). Rewrite external users:

```bash
rewrite 'usecase\.TrustService' 'trust.TrustService'
rewrite 'usecase\.TrustConfig' 'trust.TrustConfig'
rewrite 'usecase\.NewPool' 'trust.NewPool'
rewrite 'usecase\.IdentityResolver' 'trust.IdentityResolver'
```

Users: `worker/core.go`, `worker/application/usecase/{order,contact,piecestore,trashchore}.go`, tests. Add `aioz-depin/worker/trust` imports where the build demands.

- [ ] **Step 3: Move the CoordDB types**

```bash
git mv worker/domain/entity/peer.go worker/trust/coorddb.go
sed -i 's|^package entity$|package trust|' worker/trust/coorddb.go
rewrite 'entity\.CoordDB' 'trust.CoordDB'
rewrite 'entity\.CoordStatus' 'trust.CoordStatus'
rewrite 'entity\.ExitProgress' 'trust.ExitProgress'
grep -rn 'entity\.Peer\b' --include='*.go' .   # rewrite each hit to trust.Peer by hand (avoid matching entity.PeerX)
```

Implementer: `worker/db/coord.go` (`CoordRepository` returns `entity.CoordDB` → now `trust.CoordDB`).

- [ ] **Step 4: Gate passes**
- [ ] **Step 5: Commit (ASK USER FIRST)** — `refactor(worker): create trust subsystem`

### Task 9: orders subsystem

`worker/application/usecase/order.go` duplicates `worker/infrastructure/order/service.go` (same `ArchivedInfo`, `Status`, `ArchiveRequest`, `DB`, `Config`/`OrderConfig`, `Service`/`OrderService`). The usecase copy is the live one (`core.go` wires `usecase.NewOrderService`); the infrastructure `Service` is dead.

**Files:**
- Move: `worker/infrastructure/order/` → `worker/orders/` (package `order` → `orders`; note plural, matching its import alias in core.go)
- Delete: `worker/orders/service.go` + `worker/orders/service_test.go` (dead duplicate — verify first)
- Move: `worker/application/usecase/order.go` → `worker/orders/service.go` (package `usecase` → `orders`)

- [ ] **Step 1: Verify the infrastructure Service is dead**

```bash
grep -rn 'order\.NewService\|order\.Service{' --include='*.go' . | grep -v 'infrastructure/order'
```

Expected: no hits. (If there are hits, STOP — report to user; the dedup decision needs revisiting.)

- [ ] **Step 2: Move, dedup, merge**

```bash
git mv worker/infrastructure/order worker/orders
git rm worker/orders/service.go worker/orders/service_test.go worker/orders/db_test.go
sed -i 's|^package order$|package orders|' worker/orders/*.go worker/orders/ordersfile/*.go 2>/dev/null; true
git mv worker/application/usecase/order.go worker/orders/service.go
sed -i 's|^package usecase$|package orders|' worker/orders/service.go
rewrite 'aioz-depin/worker/infrastructure/order' 'aioz-depin/worker/orders'
```

Check each remaining `worker/orders/*.go` actually had `package order` (ordersfile/ has its own package — leave it). In `worker/orders/service.go` remove the self-import (`orders "aioz-depin/worker/orders"`) and the `orders.` qualifier on `FileStore`. Note `db_test.go` tests the deleted duplicate's `DB` interface usage — if it tests the *kept* types instead, keep it (build will tell).

- [ ] **Step 3: Rewrite external users**

```bash
rewrite 'usecase\.OrderService' 'orders.OrderService'
rewrite 'usecase\.OrderConfig' 'orders.OrderConfig'
rewrite 'usecase\.NewOrderService' 'orders.NewOrderService'
```

Users: `worker/core.go` (also already imports `orders` alias for FileStore — now the same package).

- [ ] **Step 4: Gate passes**
- [ ] **Step 5: Commit (ASK USER FIRST)** — `refactor(worker): create orders subsystem, remove dead duplicate order service`

### Task 10: pieces package + pkg/filewalker

Split `worker/infrastructure/piece` into the piece store layer (`worker/pieces/`, spec amendment #3) and the walkers (`worker/pkg/filewalker/`).

**Files:**
- Move: `worker/infrastructure/piece/{store.go,store_test.go,cache.go,cache_test.go,expired_info.go,pieceexpiration.go,pieceexpiration_unix.go,pieceexpiration_windows.go,readwrite.go,stat.go}` → `worker/pieces/` (package `piece` → `pieces`)
- Move: `worker/infrastructure/piece/{filewalker.go,filewalkerdb.go}` + `worker/infrastructure/piece/lazyfilewalker/` → `worker/pkg/filewalker/` (package `piece` → `filewalker`; `lazyfilewalker` keeps its name as subpackage)

- [ ] **Step 1: Move the store layer**

```bash
mkdir -p worker/pieces
for f in store.go store_test.go cache.go cache_test.go expired_info.go pieceexpiration.go pieceexpiration_unix.go pieceexpiration_windows.go readwrite.go stat.go; do git mv "worker/infrastructure/piece/$f" "worker/pieces/$f"; done
sed -i 's|^package piece$|package pieces|;s|^package piece_test$|package pieces_test|' worker/pieces/*.go
```

- [ ] **Step 2: Move the walkers**

```bash
mkdir -p worker/pkg/filewalker
git mv worker/infrastructure/piece/filewalker.go worker/pkg/filewalker/filewalker.go
git mv worker/infrastructure/piece/filewalkerdb.go worker/pkg/filewalker/filewalkerdb.go
git mv worker/infrastructure/piece/lazyfilewalker worker/pkg/filewalker/lazyfilewalker
sed -i 's|^package piece$|package filewalker|' worker/pkg/filewalker/*.go
rmdir worker/infrastructure/piece
```

- [ ] **Step 3: Stitch the split**

The two halves referenced each other inside one package; now they need imports. Rewrite paths first:

```bash
rewrite 'aioz-depin/worker/infrastructure/piece/lazyfilewalker' 'aioz-depin/worker/pkg/filewalker/lazyfilewalker'
rewrite 'aioz-depin/worker/infrastructure/piece' 'aioz-depin/worker/pieces'
rewrite 'piece\.Store' 'pieces.Store'
rewrite 'piece\.NewStore' 'pieces.NewStore'
rewrite 'piece\.Config' 'pieces.Config'
rewrite 'piece\.FileWalker' 'filewalker.FileWalker'
rewrite 'piece\.NewFileWalker' 'filewalker.NewFileWalker'
```

Then `go build ./worker/...` and resolve the remaining cross-references by hand: symbols that `filewalker.go` used from store files (or vice versa) must be imported with their new qualifier (`pieces.X` from filewalker, `filewalker.X` from pieces — only `pieces` → `filewalker` direction is allowed; if filewalker needs a `pieces` type, that type moves to `filewalker` (e.g. shared DB interfaces like `GCFilewalkerProgressDB` live in `filewalkerdb.go`) and `pieces`/`worker/db` reference it from there). Users to recheck: `worker/application/usecase/{retain,trashchore,piecestore}.go`, `worker/infrastructure/retain/{retain,runner}.go`, `worker/db/piece_repos.go`, `worker/core.go`.

- [ ] **Step 4: Gate passes**
- [ ] **Step 5: Commit (ASK USER FIRST)** — `refactor(worker): split piece into pieces store layer and pkg/filewalker`

### Task 11: retain subsystem

**Files:**
- Move: `worker/infrastructure/retain/` → `worker/retain/` (all files, package name unchanged)
- Move: `worker/application/usecase/retain.go` → `worker/retain/runner_service.go` (package → `retain`; type `RetainService` kept — distinct from existing `retain.Service`)
- Move: `worker/application/usecase/trashchore.go` → `worker/retain/chore.go` (package → `retain`)

- [ ] **Step 1: Move and merge**

```bash
git mv worker/infrastructure/retain worker/retain
git mv worker/application/usecase/retain.go worker/retain/runner_service.go
git mv worker/application/usecase/trashchore.go worker/retain/chore.go
sed -i 's|^package usecase$|package retain|' worker/retain/runner_service.go worker/retain/chore.go
rewrite 'aioz-depin/worker/infrastructure/retain' 'aioz-depin/worker/retain'
rewrite 'usecase\.RetainService' 'retain.RetainService'
rewrite 'usecase\.NewRetainService' 'retain.NewRetainService'
rewrite 'usecase\.RetainConfig' 'retain.RetainConfig'
rewrite 'usecase\.TrashChore' 'retain.TrashChore'
rewrite 'usecase\.NewTrashChore' 'retain.NewTrashChore'
```

In the two moved files: remove self-imports of `aioz-depin/worker/retain` and drop `retain.` qualifiers. `chore.go` references `*TrustService` — it now needs `aioz-depin/worker/trust` import and `*trust.TrustService` (allowed: Chore → Service cross-subsystem via Service).

- [ ] **Step 2: Gate passes** (collision check: `RetainConfig` vs existing `retain.Config` — different names, no clash; build confirms)
- [ ] **Step 3: Commit (ASK USER FIRST)** — `refactor(worker): create retain subsystem`

### Task 12: piecestore subsystem

**Files:**
- Move: `worker/infrastructure/piecestore/` → `worker/piecestore/` (package name unchanged: `backend.go`, `backend_test.go`, `helpers.go`)
- Move: `worker/application/usecase/piecestore.go` → `worker/piecestore/service.go` (package → `piecestore`)
- Move: `worker/presentation/grpc/piecestore.go` → `worker/piecestore/endpoint.go` (package `grpc` → `piecestore`)

- [ ] **Step 1: Move and merge**

```bash
git mv worker/infrastructure/piecestore worker/piecestore
git mv worker/application/usecase/piecestore.go worker/piecestore/service.go
git mv worker/presentation/grpc/piecestore.go worker/piecestore/endpoint.go
sed -i 's|^package usecase$|package piecestore|' worker/piecestore/service.go
sed -i 's|^package grpc$|package piecestore|' worker/piecestore/endpoint.go
rewrite 'aioz-depin/worker/infrastructure/piecestore' 'aioz-depin/worker/piecestore'
rewrite 'usecase\.PieceStoreService' 'piecestore.PieceStoreService'
rewrite 'usecase\.NewPieceStoreService' 'piecestore.NewPieceStoreService'
rewrite 'usecase\.PieceStoreConfig' 'piecestore.PieceStoreConfig'
rewrite 'usecase\.QueueRetain' 'piecestore.QueueRetain'
rewrite 'usecase\.RestoreTrash' 'piecestore.RestoreTrash'
rewrite 'grpc\.NewPieceStoreEndpoint' 'piecestore.NewPieceStoreEndpoint'
rewrite 'grpc\.PieceStoreEndpoint' 'piecestore.PieceStoreEndpoint'
```

In the moved files: remove self-imports (`piecestore "aioz-depin/worker/infrastructure/piecestore"`, `usecase` import in endpoint.go) and drop self-qualifiers. `endpoint.go` keeps `PingStatsSource` references — that interface is defined in `worker/presentation/grpc` (`grpc.PingStatsSource`); move its declaration from `worker/presentation/grpc/` into `worker/piecestore/endpoint.go` if it lives in `piecestore.go`'s file, otherwise leave for Task 14 (contact) and qualify as needed (build will locate it: `grep -rn 'PingStatsSource' worker/presentation/grpc/`).

- [ ] **Step 2: Cycle check** — `piecestore` imports `retain`, `pieces`, `trust`, `pkg/hashstore`; verify `retain` does NOT import `piecestore`:

```bash
grep -rn 'aioz-depin/worker/piecestore' worker/retain/
```

Expected: no hits. (`QueueRetain`/`RestoreTrash` are consumer-side interfaces in `piecestore` — `retain` types satisfy them without importing `piecestore`.)

- [ ] **Step 3: Gate passes**
- [ ] **Step 4: Commit (ASK USER FIRST)** — `refactor(worker): create piecestore subsystem`

### Task 13: monitor subsystem

**Files:**
- Move: `worker/application/usecase/monitor.go` → `worker/monitor/service.go` (package → `monitor`)
- Move: `worker/application/usecase/speedtest.go` → `worker/monitor/speedtest.go` (package → `monitor`)
- Move: `worker/infrastructure/space.go` → `worker/monitor/spacereport.go` (interface `SpaceReport`, package `infrastructure` → `monitor`)
- Move: `worker/infrastructure/systeminfo.go` → `worker/monitor/sysinforeport.go` (interface `SystemInfoReporter`, package → `monitor`)
- Move: `worker/infrastructure/reporter.go` → `worker/monitor/reporter.go`; `worker/infrastructure/runner.go` → `worker/monitor/runner.go` (package → `monitor`)
- Move: `ports.SystemInfoProvider` (from `worker/application/usecase/ports/system_info_provider.go`) → `worker/pkg/sysinfo/provider.go` as `sysinfo.Provider`
- Move: `ports.ResultReporter` (`ports/reporter.go`) and `ports.MeasureRunner` (`ports/runner.go`) → `worker/monitor/speedtest.go` (consumer-side)
- Move: `worker/domain/valueobjects/speedtest.go` → `worker/monitor/speedtest_result.go` (package → `monitor`)

- [ ] **Step 1: Move files**

```bash
mkdir -p worker/monitor
git mv worker/application/usecase/monitor.go worker/monitor/service.go
git mv worker/application/usecase/speedtest.go worker/monitor/speedtest.go
git mv worker/infrastructure/space.go worker/monitor/spacereport.go
git mv worker/infrastructure/systeminfo.go worker/monitor/sysinforeport.go
git mv worker/infrastructure/reporter.go worker/monitor/reporter.go
git mv worker/infrastructure/runner.go worker/monitor/runner.go
git mv worker/domain/valueobjects/speedtest.go worker/monitor/speedtest_result.go
sed -i 's|^package usecase$|package monitor|;s|^package infrastructure$|package monitor|;s|^package valueobjects$|package monitor|' worker/monitor/*.go
```

- [ ] **Step 2: Relocate the ports interfaces**

1. Create `worker/pkg/sysinfo/provider.go` containing `package sysinfo` and the interface from `ports/system_info_provider.go` renamed `SystemInfoProvider` → `Provider` (methods unchanged: `GetOSInfo`, `GetHardwareInfo`). Delete `ports/system_info_provider.go`.
2. Paste `ResultReporter` and `MeasureRunner` interface declarations into `worker/monitor/speedtest.go`; delete `ports/reporter.go` and `ports/runner.go`.

```bash
rewrite 'ports\.SystemInfoProvider' 'sysinfo.Provider'
rewrite 'ports\.ResultReporter' 'monitor.ResultReporter'
rewrite 'ports\.MeasureRunner' 'monitor.MeasureRunner'
rewrite 'infrastructure\.SpaceReport' 'monitor.SpaceReport'
rewrite 'infrastructure\.SystemInfoReporter' 'monitor.SystemInfoReporter'
rewrite 'infrastructure\.NewReporter' 'monitor.NewReporter'
rewrite 'infrastructure\.NewRunner' 'monitor.NewRunner'
rewrite 'usecase\.MonitorService' 'monitor.MonitorService'
rewrite 'usecase\.NewMonitorService' 'monitor.NewMonitorService'
rewrite 'usecase\.MonitorConfig' 'monitor.MonitorConfig'
rewrite 'usecase\.SpeedtestUseCase' 'monitor.SpeedtestUseCase'
rewrite 'usecase\.NewSpeedtestUseCase' 'monitor.NewSpeedtestUseCase'
rewrite 'usecase\.DiskVerification' 'monitor.DiskVerification'
rewrite 'valueobjects\.SpeedTest' 'monitor.SpeedTest'
```

Inside `worker/monitor/*.go` drop self-qualifiers/self-imports as before. `pkg/sysinfo/systeminfo.go` (implementation) must satisfy `sysinfo.Provider` — same methods, just verify it compiles. `worker/pkg/space/shared.go` (`NewSharedDisk`) returns the implementation consumed as `monitor.SpaceReport` — since the *consumer* holds the interface, `space` needs no import of `monitor` (returns concrete type or takes the interface from caller; if `space` currently references `infrastructure.SpaceReport` in its signature, change it to return the concrete `*SharedDisk` and let `core.go` assign it to the `monitor.SpaceReport`-typed field).

⚠️ `monitor.MonitorService` depends on `*ContactService` (still in `usecase` until Task 14). The moved `service.go` keeps importing `aioz-depin/worker/application/usecase` for it temporarily — that's fine; Task 14 cleans it.

- [ ] **Step 3: Gate passes**
- [ ] **Step 4: Commit (ASK USER FIRST)** — `refactor(worker): create monitor subsystem (incl. speedtest), dissolve remaining ports interfaces`

### Task 14: contact subsystem (incl. Contract→Contact naming fix)

**Files:**
- Move: `worker/application/usecase/contact.go` → `worker/contact/service.go` (package → `contact`)
- Move: `worker/application/usecase/worker_info.go` → `worker/contact/register.go` (package → `contact`; the Core path's hub registration)
- Move: `worker/presentation/grpc/contact.go` → `worker/contact/endpoint.go`; `worker/presentation/grpc/mon.go` → `worker/contact/mon_endpoint.go` (or merge `mon` vars; build decides)
- Move: `worker/presentation/chore/contact.go` → `worker/contact/chore.go`; `worker/presentation/chore/mon.go` → merge likewise
- Move: `worker/utils/pingstat/` → `worker/contact/pingstat.go` (package → `contact`) — also resolves `PingStatsSource` (used by piecestore endpoint; it stays an interface where it's declared — check Task 12 note)
- Move: `worker/domain/entity/worker.go` (`WorkerInfo`) → `worker/contact/workerinfo.go`; `worker/domain/entity/node_info.go` → `worker/contact/nodeinfo.go`

- [ ] **Step 1: Move files**

```bash
mkdir -p worker/contact
git mv worker/application/usecase/contact.go worker/contact/service.go
git mv worker/application/usecase/worker_info.go worker/contact/register.go
git mv worker/presentation/grpc/contact.go worker/contact/endpoint.go
git mv worker/presentation/chore/contact.go worker/contact/chore.go
git mv worker/utils/pingstat/pingstat.go worker/contact/pingstat.go
git mv worker/domain/entity/worker.go worker/contact/workerinfo.go
git mv worker/domain/entity/node_info.go worker/contact/nodeinfo.go
sed -i 's|^package usecase$|package contact|;s|^package grpc$|package contact|;s|^package chore$|package contact|;s|^package pingstat$|package contact|;s|^package entity$|package contact|' worker/contact/*.go
```

`presentation/grpc/mon.go` and `presentation/chore/mon.go`: each defines package-level monkit plumbing for its old package. After the piecestore endpoint moved (Task 12) and contact moved here, check what remains in `worker/presentation/` — move each `mon.go`'s needed declarations into the new packages (a `var mon = monkit.Package()` per package), then delete the leftover files and the `presentation/` tree:

```bash
git rm worker/presentation/grpc/mon.go worker/presentation/chore/mon.go
rmdir worker/presentation/grpc worker/presentation/chore worker/presentation
```

(If Task 12's endpoint.go referenced `mon` from the old grpc package, it needs its own `var mon = monkit.Package()` — add it there when this delete breaks the build.)

- [ ] **Step 2: Rewrite users + Contract→Contact rename**

```bash
rewrite 'usecase\.ContactService' 'contact.Service'
rewrite 'usecase\.NewContractService' 'contact.NewService'
rewrite 'usecase\.ContractConfig' 'contact.Config'
rewrite 'usecase\.WorkerInfoUseCase' 'contact.WorkerInfoUseCase'
rewrite 'usecase\.NewWorkerInfoUseCase' 'contact.NewWorkerInfoUseCase'
rewrite 'grpc\.ContactEndpoint' 'contact.Endpoint'
rewrite 'grpc\.NewContactEndpoint' 'contact.NewEndpoint'
rewrite 'chore\.ContactChore' 'contact.Chore'
rewrite 'chore\.NewChore' 'contact.NewChore'
rewrite 'pingstat\.NewPingStatsSource' 'contact.NewPingStatsSource'
rewrite 'entity\.WorkerInfo' 'contact.WorkerInfo'
```

Then inside `worker/contact/*.go`: rename the declarations to match (`ContactService` → `Service`, `NewContractService` → `NewService`, `ContractConfig` → `Config`, `ContactEndpoint` → `Endpoint`, `ContactChore` → `Chore`), drop self-imports/qualifiers. In `worker/core.go`: rename the field `Contract` → `Contact` and `Config.Contract` → `Config.Contact` (grep `config.Contract\.` for users — also `cmd/worker` config binding and `dev/worker1` config keys if the cfg struct tag changes; do NOT change struct tags/YAML keys in this task — only Go identifiers — so runtime config files keep working).

- [ ] **Step 3: Gate passes**
- [ ] **Step 4: Commit (ASK USER FIRST)** — `refactor(worker): create contact subsystem, fix Contract/Contact naming`

### Task 15: version subsystem

**Files:**
- Move: `worker/application/usecase/autoupdate.go` + `autoupdate_test.go` → `worker/version/service.go` / `service_test.go` (package → `version`)
- Move: `worker/infrastructure/versioncontrol/{http_gateway.go,http_gateway_test.go,version_provider.go}` → `worker/version/` (package `versioncontrol` → `version`)
- Move: `worker/application/dto/version.go` → `worker/version/dto.go` (package → `version`)
- Move: `worker/domain/valueobjects/version.go` (`CheckVersionResult`) → `worker/version/checkresult.go`
- Move: `ports.AutoUpdater`, `ports.VersionProvider`, `ports.VersionControlGateway` (from `ports/auto_updater.go`, `ports/version_provider.go`) → `worker/version/` (consumer-side, into `service.go`)

⚠️ Import-alias note: root `pkg/version` exists. Any file importing both uses an alias: `workerversion "aioz-depin/worker/version"`.

- [ ] **Step 1: Move and merge**

```bash
mkdir -p worker/version
git mv worker/application/usecase/autoupdate.go worker/version/service.go
git mv worker/application/usecase/autoupdate_test.go worker/version/service_test.go
git mv worker/infrastructure/versioncontrol/http_gateway.go worker/version/http_gateway.go
git mv worker/infrastructure/versioncontrol/http_gateway_test.go worker/version/http_gateway_test.go
git mv worker/infrastructure/versioncontrol/version_provider.go worker/version/version_provider.go
git mv worker/application/dto/version.go worker/version/dto.go
git mv worker/domain/valueobjects/version.go worker/version/checkresult.go
sed -i 's|^package usecase$|package version|;s|^package versioncontrol$|package version|;s|^package dto$|package version|;s|^package valueobjects$|package version|' worker/version/*.go
rmdir worker/infrastructure/versioncontrol worker/application/dto
```

Paste the three interface declarations from `ports/auto_updater.go` and `ports/version_provider.go` into `worker/version/service.go`; delete those two ports files (`ports/` should now be empty — `rmdir worker/application/usecase/ports`).

- [ ] **Step 2: Rewrite users**

```bash
rewrite 'aioz-depin/worker/infrastructure/versioncontrol' 'aioz-depin/worker/version'
rewrite 'ports\.AutoUpdater' 'version.AutoUpdater'
rewrite 'ports\.VersionProvider' 'version.VersionProvider'
rewrite 'ports\.VersionControlGateway' 'version.VersionControlGateway'
rewrite 'usecase\.AutoUpdate' 'version.AutoUpdate'
rewrite 'usecase\.NewAutoUpdate' 'version.NewAutoUpdate'
rewrite 'dto\.' 'version.'
rewrite 'valueobjects\.CheckVersionResult' 'version.CheckVersionResult'
rewrite 'versioncontrol\.NewVersionProvider' 'version.NewVersionProvider'
```

(`rewrite 'dto\.' 'version.'` is broad — verify hits first with `grep -rn 'dto\.' --include='*.go' worker cmd`; only worker's `application/dto` users should match.) Fix aliases where `pkg/version` is co-imported (check `worker/core.go`, `worker/di/provider.go`, `cmd/worker/*`). Drop self-imports/qualifiers in moved files.

- [ ] **Step 3: Gate passes** (also: `ls worker/domain/valueobjects` → only `authheader.go` and `systeminfo.go` remain)
- [ ] **Step 4: Commit (ASK USER FIRST)** — `refactor(worker): create version subsystem`

### Task 16: Peer wiring — peer.go, hand-wired Core, delete di/

**Files:**
- Rename: `worker/core.go` → `worker/peer.go`; type `API` → `Peer`, `NewAPI` → `NewPeer`
- Create: `worker/core_wire.go` — hand-wired `NewCore` (transplanted from `worker/di/wire_gen.go` + `provider.go`)
- Delete: `worker/di/`
- Modify: `cmd/worker/api.go`, `cmd/worker/helpers.go`, `cmd/worker/main.go`

- [ ] **Step 1: Rename API → Peer**

```bash
git mv worker/core.go worker/peer.go
rewrite 'worker\.NewAPI' 'worker.NewPeer'
rewrite 'worker\.API' 'worker.Peer'
sed -i 's|type API struct|type Peer struct|;s|func NewAPI(|func NewPeer(|;s|api := API{|api := Peer{|;s|\*API)|\*Peer)|g' worker/peer.go
go build ./worker/... ./cmd/worker/...   # fix any receiver/reference the seds missed
```

Restructure the `Peer` struct fields to group by subsystem (matching the spec's peer shape): keep existing nested structs but they're now `Contact struct{ Service *contact.Service; Endpoint *contact.Endpoint; Chore *contact.Chore }` etc. — this is renaming-only; construction order in `NewPeer` stays byte-for-byte equivalent to the current `NewAPI`.

- [ ] **Step 2: Hand-wire Core**

Create `worker/core_wire.go` (`package worker`) containing `func NewCore(ctx context.Context, logger *zap.Logger, cfg *config.AppConfig, privKey []byte) (*Core, func(), error)`. Its body is the function body of `InitializeWorker` from `worker/di/wire_gen.go` copied verbatim, with the provider helper functions (`NewCircuitBreaker`, `NewRetrier`, `NewKeyLibp2p`, `NewP2PNode`, `NewP2PClient`, `NewKeyTool`, `NewReporter`, `NewRunner`, `NewDiskUsage`, `NewFileManager`, `ProvideCronRunners`, `NewVersionControllerGateway`, `NewAutoUpdater`) moved in from `worker/di/provider.go` (lowercase them — they're now package-internal: `newCircuitBreaker`, etc.). Update `cmd/worker/helpers.go`: `di.InitializeWorker(...)` → `worker.NewCore(...)`; drop the `aioz-depin/worker/di` import.

- [ ] **Step 3: Delete di and the Core struct's old home references**

```bash
git rm -r worker/di
go build ./... 
```

- [ ] **Step 4: Behavioral check** — construction order in `NewCore` must match `wire_gen.go` exactly (diff the call sequence by eye); cleanup function composition must be preserved.

- [ ] **Step 5: Gate passes**
- [ ] **Step 6: Commit (ASK USER FIRST)** — `refactor(worker): hand-wire peer and core, remove google/wire`

### Task 17: Dissolve leftovers, delete empty trees

**Added during execution (Task 15 quality review):** Pre-existing bug surfaced: `worker/version/service.go` `sanitizeError` redacts key NAMES but not VALUES (`signature=deadbeef` → `[redacted]=deadbeef`), so secrets still leak into logs — its test fails since baseline. Also `CheckVersionResponse` in `worker/version/dto.go` is dead code. Both for the Task 18 report / follow-up fix, NOT for a mechanical commit.

**Added during execution (Task 11 spec review):** `worker/retain` now contains two parallel near-duplicate service types: `Config`/`Service`/`NewService` (ex-infrastructure queue service, used by tests + piecestore usecase) and `RetainConfig`/`RetainService`/`NewRetainService` (ex-usecase wrapper, wired in core.go), with identical config fields. Plan-endorsed coexistence for the mechanical phase. Consolidation decision (likely: keep `Service`, alias or delete `RetainService`) goes to the user at final review — record in Task 18 report.

**Added during execution (Task 6 quality review):** `pkg/key/` at the repo root is a byte-for-byte duplicate of `worker/pkg/keytool` with ZERO importers (verified by grep at commit 3ff57e0). Step: re-verify `grep -rn '"aioz-depin/pkg/key"' --include='*.go' .` is still empty, then ASK THE USER before `git rm -r pkg/key` (root-level package deletion needs explicit confirmation). Also move the CLI flag constants (`flagPassword`, `flagMnemonicWord`, `minPasswordLength`) from `worker/pkg/keytool/dto.go` back to `cmd/keytool/command.go` where they belong.

**Files:**
- Move: `worker/config/config.go` → `worker/config.go`? — NO: `cmd/worker` imports `worker/config` and `AppConfig` would collide conceptually with `worker.Config` (Peer config). Keep `worker/config/` as-is this round (flagged for the cosmetic pass; the spec's `config.go` line is satisfied by `worker.Config` already living in `peer.go`).
- Move: `worker/domain/valueobjects/authheader.go` → consumers are `monitor/reporter.go`, `monitor/runner.go`, `version/http_gateway.go`, `pkg/sysinfo/systeminfo.go` → it's transport plumbing for coord HTTP calls: move to `worker/pkg/p2pc/authheader.go` (package `p2pc`), rewrite `valueobjects.AuthHeaders` → `p2pc.AuthHeaders`
- Move: `worker/domain/valueobjects/systeminfo.go` (`OSInfo`, `CPU`, `GPU`, `HardwareInfo`, `HardDriveInfo`) → `worker/pkg/sysinfo/types.go` (package `sysinfo`), rewrite `valueobjects.OSInfo` → `sysinfo.OSInfo` etc. (5 types)
- Move: remaining `worker/domain/entity/` files (`piece.go` with its DB interfaces/records, `monitor`-ish types `SystemInfo`, `DiskSpace`, `StorageStatus`, `MonitorInfo`, `Worker`) → owners: piece-DB types used by `worker/db` + `pieces` + `pkg/filewalker` → move `entity/piece.go` to `worker/pieces/types.go` unless `pkg/filewalker` needs them (then `worker/pkg/filewalker/types.go`; check with `grep -rln 'entity\.' worker/pkg/filewalker`); `SystemInfo`/`DiskSpace`/`StorageStatus`/`MonitorInfo` → `worker/monitor/types.go`; `Worker` → check users with `grep -rn 'entity\.Worker\b'` and move beside its main consumer
- Delete: `worker/application/`, `worker/domain/`, `worker/infrastructure/`, `worker/utils/` (all should be empty or near-empty now)

- [ ] **Step 1: Move authheader and sysinfo VOs**

```bash
git mv worker/domain/valueobjects/authheader.go worker/pkg/p2pc/authheader.go
git mv worker/domain/valueobjects/systeminfo.go worker/pkg/sysinfo/types.go
sed -i 's|^package valueobjects$|package p2pc|' worker/pkg/p2pc/authheader.go
sed -i 's|^package valueobjects$|package sysinfo|' worker/pkg/sysinfo/types.go
rewrite 'valueobjects\.AuthHeaders' 'p2pc.AuthHeaders'
for t in OSInfo CPU GPU HardwareInfo HardDriveInfo; do rewrite "valueobjects\\.$t" "sysinfo.$t"; done
```

- [ ] **Step 2: Move remaining entity types per the mapping above**

For each remaining file in `worker/domain/entity/`: `grep -rn 'entity\.<Type>' --include='*.go' .` to confirm the consumer set matches the mapping, `git mv` to the owner package, fix package clause, rewrite qualifier. If a type has zero users, delete it instead of moving (report which in the commit message).

- [ ] **Step 3: Delete the empty DDD trees**

```bash
find worker/application worker/domain worker/infrastructure worker/utils -name '*.go' | grep .   # expect: NO output; investigate any stragglers before deleting
git rm -r worker/application worker/domain worker/infrastructure worker/utils 2>/dev/null
rmdir worker/presentation 2>/dev/null; true
```

- [ ] **Step 4: Gate passes**
- [ ] **Step 5: Commit (ASK USER FIRST)** — `refactor(worker): dissolve entity/valueobjects remnants, remove DDD folder tree`

### Task 18: Final audit

- [ ] **Step 1: Full build, vet, test**

```bash
go build ./... && go vet ./... && go test ./worker/... ./cmd/worker/... ./cmd/keytool/...
```

Expected: PASS, modulo the pre-existing failures recorded in Task 0 (`/tmp/worker-test-baseline.txt`).

- [ ] **Step 2: Layer-rule audit** (each must print nothing):

```bash
# pkg never imports subsystems:
grep -rn 'aioz-depin/worker/\(contact\|piecestore\|orders\|trust\|retain\|monitor\|version\|pieces\|db\|server\)"' worker/pkg/
# db imports no subsystem services (interfaces only — trust.CoordDB etc. are allowed; flag anything importing endpoint/chore symbols):
grep -rn 'Endpoint\|Chore' worker/db/*.go
# no leftover DDD imports anywhere:
grep -rn 'worker/application\|worker/domain\|worker/infrastructure\|worker/presentation\|worker/di' --include='*.go' .
```

- [ ] **Step 3: Lint**

Run: `make lint LINT_TARGET=./worker/...`
Expected: no new findings vs baseline (run on the base branch first if unsure).

- [ ] **Step 4: GitNexus** (per CLAUDE.md, if the MCP tools are available in the session): run `npx gitnexus analyze`, then `gitnexus_detect_changes()` and confirm affected flows are confined to worker + cmd/worker + cmd/keytool + pkg/key. If unavailable, state so and rely on Steps 1–3.

- [ ] **Step 5: Integration smoke test** — run the worker against the dev setup (`dev/worker1/` config, per project convention) and verify it starts, connects, and serves: ask the user for the exact run command they use if not obvious from `cmd/worker`.

- [ ] **Step 6: Commit any audit fixes (ASK USER FIRST)**, then hand back to the user for branch review. Do NOT merge or push.
