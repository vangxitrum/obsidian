---
type: spec
project: depin
tags: [depin, worker, alias, bigint, schema]
created: 2026-06-04
status: implemented
---

# Worker Alias (bigint) + legacy worker-table retirement

Date: 2026-06-04
Status: implemented

## Problem

Persisted piece info is heavy: every piece reference carries a 16-byte worker
UUID (36+ chars serialized in `segments.order_limits` JSON) plus, inside stored
order limits, the worker address. To shrink piece info we need a compact
numeric worker identifier — like Storj's node alias — that piece records can
reference instead of the UUID/address.

Separately, the coordinator still carried the retired peer_id-keyed `workers`
table (and the dormant code stack that used it), while the live table was named
`workers_info`.

## Decisions (user-approved)

1. **Create the alias only** for now. Piece-info structures
   (`segments.order_limits`, `segment_piece_uploads`) are NOT changed yet;
   replacing the worker UUID there is a follow-up.
2. Alias is a **column on the worker table**, assigned by the database
   (sequence) the first time a worker registers via Contact.CheckIn. Not added
   to any proto — wire protocol unchanged (uplink still needs UUID + address in
   order limits to dial workers).
3. **Drop the legacy `workers` table** (and `reputations`, FK-dependent) and
   **delete the dormant legacy code** that used it.
4. **Rename `workers_info` → `workers`** (plural, matching other tables).

## Design

### Alias column (`coord/domain/entity/worker.go`)

```go
Alias int64 `gorm:"column:alias;not null;autoIncrement;uniqueIndex;<-:false"`
```

- `autoIncrement` (not an explicit `type:bigserial`): the gorm postgres driver
  creates/owns the backing sequence; an explicit bigserial type tag makes later
  AutoMigrate runs emit invalid `ALTER COLUMN ... TYPE bigserial` (verified).
- `<-:false`: gorm omits the column from INSERT and UPDATE. The sequence
  default assigns it on first insert; the `AddWorker` upsert (`db.Save`) on
  re-check-in can never clobber it. No repository changes were needed.
- `TableName()` → `"workers"`.

### Migration `000006_worker_alias`

Up: `DROP TABLE IF EXISTS reputations` (FK first), `DROP TABLE IF EXISTS
workers` (legacy), then rename `workers_info` → `workers` guarded by
`IF EXISTS` — on fresh databases `workers_info` never exists and gorm
AutoMigrate creates `workers` directly from the entity. The alias column itself
is owned by AutoMigrate (adds it to pre-existing tables and backfills rows from
the sequence). Down: rename back and recreate the legacy DDL from 000001.

### Legacy deletion closure

Deleted (compiled but unreachable from the live `coord api` command):

- `cmd/coord/main.go`: `run` command, p2p node setup, `getPrivKeyFromBytes`,
  `isSetup`, `Marshaller`
- `coord/core.go`, `coord/di/`, `coord/presentation/api/p2p/`,
  `coord/application/services/p2pwatcher/`, `coord/infrastructure/scheduler/`
- `coord/application/usecase/worker.go` (WorkerManagerUseCase),
  `uptime_score_usecase.go`
- `coord/domain/entity/`: `worker_old.go`, `worker_builder.go`,
  `reputation.go`, `cpu_array.go`, `gpu_array.go`
- Legacy repo methods (`GetWorker`, `UpsertWorker`, `GetWorkersLimit`,
  `UpdateBatchWorker`) and their interface entries

Retargeted instead of deleted: `worker_filters.go` and the `WorkerFilter`
interface now take `*Worker`. The leaf filters (country, reputation) return
`true` with a TODO — the new check-in schema carries no geo/reputation
metadata, and the live selection path does not apply filters (see TODO in
`WorkerSelectionUseCase.SelectWorkers`). Keeping the types means serialized
`WorkerFilterWrapper` values in `placements` rows still load.

## Verification (performed against the live dev DB)

- `go build` / `go vet` / `go test ./coord/...` clean (uplink/ module breakage
  is pre-existing and unrelated).
- Migration applied on the dev DB: `workers` renamed, legacy tables gone,
  `schema_migrations` at version 6 clean.
- Live upsert test through the real repository: new worker gets `alias` from
  `workers_alias_seq`; a second `AddWorker` (re-check-in) updates the address
  but preserves the alias; existing registered worker was backfilled with
  `alias = 1`.
- `dev/coord/.air.toml` runs `api` — nothing referenced the removed `run`
  command. `uplink-sdk/test/integration_test.go` updated to query `workers`.

## Follow-up (next step, separate change)

Replace the worker UUID in persisted piece info with the alias: store
`worker_alias` in `segments.order_limits` records (and optionally
`segment_piece_uploads`), resolving UUID/address via the `workers` table when
constructing wire order limits.
