# Vertical Split Restructure — Design

**Date:** 2026-06-04
**Scope:** `worker/`, `coord/`, `versioncontrol/` (in that order). Explicitly out of scope: `uplink/`, `uplink-sdk/` (import-path fixes only where they reference moved packages).
**Reference:** `SOURCE_STRUCTURE.md` (layered peer architecture: Peer → Subsystem → Endpoint/Chore/Service → Store).

## Goal

Replace the current DDD/clean-architecture layout (`domain/`, `application/usecase/`, `infrastructure/`, `presentation/`) with a vertical split: one folder per subsystem, each containing `Endpoint` (gRPC/HTTP), `Service` (business logic), `Store` (DB access), and `Chore` (background loops), wired by a hand-written Peer.

## Decisions

| Decision             | Choice                                                                                                            |
| -------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Components & order   | worker → coord → versioncontrol                                                                                   |
| Approach             | Restructure + light cleanup (collapse usecase→ports→infrastructure indirection; no behavior changes)              |
| DI                   | Hand-wired `peer.go`; delete `worker/di/` (google/wire)                                                           |
| Storage engines      | `worker/pkg/` (hashstore, blobstore, filewalker, etc.)                                                            |
| Package layout       | Flat: one Go package per subsystem (`contact.Service`, `contact.Endpoint`, …)                                     |
| Store implementation | Interface in subsystem (`contact.Store`); SQL implementations + migrations centralized in component `db/` package |
| Migration strategy   | Subsystem-by-subsystem; build green + tests pass + one commit per step                                            |

## Layer rules (from SOURCE_STRUCTURE.md)

- Endpoint → Service → Store; Chore → Service; Service → Service (cross-subsystem allowed)
- Forbidden: Store → anything above it, Service → Endpoint/Chore, Endpoint ↔ Chore, Chore ↔ Chore
- `pkg/` (root and component-local) never imports subsystem packages
- Consumer-side interfaces: each subsystem declares the small interfaces it consumes (the `application/usecase/ports/` package dissolves)

## Target structure: `worker/`

```
worker/
├── peer.go               # Peer struct + New() — wires DB → Services → Endpoints → Chores
│                         # (replaces core.go wiring + deletes worker/di/)
├── config.go             # top-level Config (from worker/config/)
│
├── contact/              # check-in with coord
│   ├── endpoint.go       #   ← presentation/grpc/contact.go
│   ├── service.go        #   ← application/usecase/contact.go + worker_info.go
│   └── chore.go          #   ← presentation/chore/contact.go
│
├── piecestore/           # piece upload/download/delete
│   ├── endpoint.go       #   ← presentation/grpc/piecestore.go
│   ├── service.go        #   ← application/usecase/piecestore.go
│   └── store.go          #   Store interface ← infrastructure/repository/piece.go
│
├── orders/               # order management & settlement
│   ├── service.go        #   ← application/usecase/order.go + infrastructure/order/service.go
│   ├── store.go          #   ← infrastructure/order/store.go (ordersfile stays as sub-helper)
│   └── chore.go          #   order sending loop
│
├── trust/                # coord trust pool
│   ├── service.go        #   ← application/usecase/trust.go + infrastructure/trust/*.go
│   └── store.go          #   ← infrastructure/repository/trust.go
│
├── retain/               # garbage collection / restore
│   ├── service.go        #   ← application/usecase/retain.go + trashchore.go
│   ├── chore.go          #   trash cleanup loop
│   └── store.go          #   ← infrastructure/retain/store.go
│
├── monitor/              # disk space, system info, health
│   ├── service.go        #   ← application/usecase/monitor.go + speedtest.go
│   └── chore.go          #   periodic reporting
│
├── version/              # version check + autoupdate
│   ├── service.go        #   ← application/usecase/autoupdate.go + infrastructure/versioncontrol/
│   └── chore.go          #   update check loop
│
├── pieces/               # piece store layer (kept separate from piecestore to break the
│   │                     # piecestore→retain→pieces import chain into a cycle-free line)
│   └── ...               #   ← infrastructure/piece/{store,cache,expiration,readwrite,stat}.go
│
├── db/                   # central DB: migrations + implements all Store interfaces
│   ├── database.go       #   ← infrastructure/db/
│   ├── pieces.go         #   implements piece DB interfaces ← infrastructure/repository/
│   └── ...
│
├── server/               # gRPC/p2p server  ← infrastructure/server/
│
└── pkg/                  # worker-local engines & libraries (no business logic)
    ├── hashstore/        #   ← infrastructure/hashstore/ (unchanged internally)
    ├── blobstore/        #   ← infrastructure/blobstore/ (incl. filestore)
    ├── filewalker/       #   ← infrastructure/piece/ walkers + lazyfilewalker
    ├── coordstore/       #   ← infrastructure/coordstore (kept separate — no coordclient merge)
    ├── p2pc/             #   ← infrastructure/p2pc (kept separate)
    ├── p2p/              #   already here
    ├── keytool/          #   ← infrastructure/key + application/usecase/key
    ├── cron/             #   ← infrastructure/cron
    ├── sysinfo/          #   ← infrastructure/systeminfo
    ├── du/, space/       #   ← infrastructure/{du,space} (kept as separate packages)
    ├── signaturecheck/   #   ← utils/signaturecheck
    └── speedestimation/  #   ← utils/speedestimation
```

Note: `worker/config/` (the `AppConfig` package for the Core path) stays as a package this round; `worker.Config` (Peer config) lives in `peer.go`. `worker.Core` (p2p registration path) is hand-wired by transplanting the generated `worker/di/wire_gen.go` body into the `worker` package.

Worker cleanups:

- `application/usecase/ports/` dissolves into consumer-side interfaces per subsystem.
- `domain/entity` + `domain/valueobjects` dissolve into owning subsystems (e.g. `entity.Piece` → `piecestore`, `entity.WorkerInfo` → `contact`). Cross-subsystem type use imports the owning package.
- `application/dto/` merges into the packages that use the types.
- Naming fix: `core.go` mixes "Contract" and "Contact" (`api.Contract.Usecase *usecase.ContactService`) — standardize on **contact**.
- `worker/utils/pingstat` joins `contact` (feeds check-in stats); `signaturecheck` and `speedestimation` move to `worker/pkg/`.

## Target structure: `coord/`

```
coord/
├── peer.go               # Peer struct + New()  ← api.go wiring
├── config.go             #   ← coord/config/
│
├── contact/              # worker check-in handling
│   ├── endpoint.go       #   ← presentation/api/grpc contact endpoint
│   ├── service.go        #   ← application/usecase/contact.go
│   └── store.go          #   Store interface ← domain/repository/worker.go
│
├── storage/              # storage offers/contracts with workers
│   ├── endpoint.go       #   ← grpc storage endpoint
│   ├── service.go        #   ← application/usecase/storage.go
│   └── store.go          #   ← domain/repository/storage.go + contract.go
│
├── file/                 # file/segment metadata (metainfo)
│   ├── endpoint.go       #   ← grpc file endpoint
│   ├── service.go        #   ← application/usecase/file.go
│   └── store.go          #   ← domain/repository/file.go + segment_piece_upload
│
├── placement/            # placement rules + worker selection
│   ├── endpoint.go       #   ← grpc placement endpoint
│   ├── service.go        #   ← application/usecase/placement.go + worker_selection.go
│   └── store.go          #   ← placement + worker repositories
│
├── client/               # client accounts
│   ├── endpoint.go       #   ← grpc client endpoint
│   ├── service.go        #   ← application/usecase/client.go
│   └── store.go          #   ← client repository
│
├── wallet/               # wallets + rewards
│   ├── service.go        #   ← application/usecase/wallet.go
│   └── store.go          #   ← wallet + reward repositories
│
├── db/                   # central DB: migrations + implements all Store interfaces
│   ├── database.go       #   ← infrastructure/db/
│   ├── contact.go        #   ← infrastructure/repository/worker.repository.go
│   ├── storage.go        #   ← infrastructure/repository/{storage,contract}.go
│   ├── file.go           #   ← infrastructure/repository/{file,segment_piece_upload}.go
│   ├── placement.go      #   ← infrastructure/repository/placement.repository.go
│   ├── client.go         #   ← infrastructure/repository/client.go
│   └── wallet.go         #   ← infrastructure/repository/{wallet,reward}.go
│
├── server/               #   ← infrastructure/server/
│
└── pkg/                  # coord-local utilities
    ├── aes/, formatter/, responses/, router/   # unchanged
    └── ratelimiter/      #   ← coord/utils/ratelimiter
```

Coord cleanups:

- **UnitOfWork dissolves.** Each subsystem gets only its own `Store` injected by the peer. Cross-store transactions become methods on the relevant Store implemented inside `coord/db/` (e.g. `file.Store.CommitSegmentWithPieces(...)`), keeping transactions in the db layer.
- **`domain/entity` dissolves** into owning subsystems: `file.go`, `segment.go`, `object.go`, `redundancy_scheme.go` → `file`; `wallet.go`, `reward_*.go`, `stripe.go` → `wallet`; `storage.go`, `contract.go` → `storage`; `worker.go` → `contact` (contact owns worker records via check-in); `worker_filters.go`, `filter_wrapper.go`, `strategies.go`, `reservoir.go` → `placement` (selection machinery). `placement` imports worker types from `contact` — a Service→Service-direction dependency, which is allowed.
- `domain/services/uptime_calculator.go` → `contact`.
- monkit instrumentation (`usecase/monkit.go`, `repository/monkit.go`) moves alongside whichever subsystem it measures.
- `presentation/api/` disappears; gRPC registration happens in each subsystem's `endpoint.go`, wired from `peer.go`.
- `coord/api_upload_integration_test.go` stays at component root, testing through the public peer.

## Target structure: `versioncontrol/`

```
versioncontrol/
├── peer.go               # Peer + New() — HTTP server lifecycle  ← api/api.go
├── config.go             #   ← versioncontrol/config/
├── endpoint.go           #   ← api/handler.go + middlewares.go (HTTP)
├── service.go            #   ← usecase/version.go
└── version.go            #   ← models/version.go + version_filter.go + api/dto/version.go
```

No Store (no DB — versions come from config) and no Chore. Domain types (`Version`, version filters) go in `version.go`; pure request/response shapes from `api/dto/` go in `endpoint.go`.

## Migration process

**Branch:** `refactor/worker-vertical-split`, based on `feat/worker-contact` — the `worker/` tree exists only on that branch (190 files vs `main`), so basing on `main` is not possible. Coord/versioncontrol phases branch from wherever this work has landed by then.

**Baseline note:** `worker/infrastructure/trust/http_source.go` has an unused import that breaks `go build ./worker/...`; fixing it is Task 0 of the worker plan.

**Each step = `go build ./...` + `go vet ./...` + component test suite green + one commit.**

1. **worker**
   1. `worker/pkg/` engines (hashstore, blobstore, filewalker, cron, sysinfo, coordclient, keytool, signaturecheck, speedestimation) — pure moves
   2. `worker/db/` + `worker/server/`
   3. Subsystems in dependency order: `trust` → `orders` → `piecestore` → `retain` → `monitor` → `contact` → `version` (trust first since piecestore/contact depend on it; contact late because it overlaps the active feature branch)
   4. `peer.go` rewrite from `core.go` + `di/wire_gen.go`; delete `worker/di/` and emptied `application/`, `domain/`, `infrastructure/`, `presentation/` folders
2. **coord**
   1. `coord/db/` + `coord/server/`
   2. Subsystems: `wallet` → `client` → `placement` → `storage` → `file` → `contact`
   3. `peer.go` from `api.go`; delete UnitOfWork and emptied DDD folders
3. **versioncontrol** — single commit

**Testing:** `*_test.go` files move with their subjects. Integration tests (`coord/api_upload_integration_test.go`, `uplink-sdk/test/integration_test.go`) are the end-to-end safety net and run unchanged. `uplink`/`uplink-sdk` receive import-path fixes only.

## Risks

- `worker` is the largest (188 files) and under active development. Mitigation: dependency-ordered moves, green build per commit, contact subsystem last among the entangled ones.
- The `peer.go` rewrite is the one step that is wiring (behavioral) rather than a move. Mitigation: construction order verified against current `di/wire_gen.go` + `core.go` line by line.
- Entangled subsystems (piecestore ↔ orders ↔ trust) may need consumer-side interfaces to avoid import cycles; this is expected and allowed by the layer rules.
- GitNexus impact analysis (per CLAUDE.md) applies during implementation; if MCP tools are unavailable in the implementation session, fall back to compiler + tests and state so.
