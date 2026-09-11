# Audit pipeline performance (2026-08-26)

Branch `feat/improve-audit`, uncommitted. The `coord audit` role was auditing
~3,300 segments/h/process and its logs were full of GORM `SLOW SQL >= 200ms` from
`coord/db/reputation_repo.go:120` and `coord/db/audit_repo.go:245`.

## The tracing technique (reusable)

SSH to the prod coordinator (`hub`) is blocked by the sandbox classifier, so the
whole diagnosis was done from Grafana's Prometheus. It works because nearly every
`coord/db` repo method and most of `coord/audit` already carries
`defer mon.Task()(&ctx)(&err)`, which exports as:

- `function_times{scope=...,name=...,kind="success",field="sum"}` - total seconds
- `function{name=...,field="total"|"highwater"}` - call count and peak concurrency

Divide `sum` by the number of segments audited
(`audit_verify_duration{field="count"}`) and you get a per-function, per-segment
cost budget with no new instrumentation. `highwater` is what revealed the
concurrency ceiling: `Verifier.Verify` was pinned at 2 while `dialFetcher.Fetch`
reached 160.

## Measured budget (per segment, 2.43 s total)

| Phase | s/segment | share |
| --- | --- | --- |
| `Verifier.Verify` (network) | 1.407 | 58% |
| `reputation.ApplyAudit` (61 serial locking txs) | 0.807 | 33% |
| `CreateAuditOrderLimits` | 0.115 | 5% |
| rest | 0.096 | 4% |

2.43 s ÷ `WorkerConcurrency=2` = 1.21 s wall/segment = ~3,000 seg/h, matching the
observed 3,279. Within verify, `Pool.get` averaged **368 ms per piece** - the coord
dialer is built with `dial.NewDefaultDialer` (no pool), so each of ~61 piece
fetches paid a full mTLS handshake.

## What shipped

1. Phase timers, skip reasons, queue-depth gauges, dial-vs-transfer split, DB pool
   stats, a `Coord Audit` Grafana dashboard, and a configurable gorm slow-query
   threshold (it was hardcoded at gorm's 200 ms default).
2. `reputation.ApplyAuditBatch` + `ReputationRepository.UpdateBatch` - one locked
   `SELECT ... FOR UPDATE` plus one `UPDATE ... FROM (VALUES ...)` for a whole
   segment's holders, replacing one transaction per holder.
3. `audit.pooled-dialer` (default off) giving the audit fetcher its own pool.
4. `NextBatch` claiming a whole round per round trip and dropping dead ids in the
   same statement; `WorkerConcurrency` 2 -> 8; reverify empty-poll backoff.
5. Migration 000050: an expression index on
   `COALESCE(last_attempt, inserted_at)` plus one on `worker_id`.

## Traps (each cost real time)

- **GORM does not scan an embedded struct** through `Raw().Scan()`, with or without
  `gorm:"embedded"`. `struct{ ID string; reputationRow }` scanned a zeroed
  `reputationRow`, so the batched read-modify-write handed the mutator a blank
  `Info` and the absolute UPDATE **reset every worker's reputation**. Fix: flatten
  the id into the row struct. Caught only by a test comparing batched vs
  per-worker results field by field.
- **`pg_class.reltuples` is -1 on a never-analyzed table.** A depth gauge built on
  the estimate reads 0 on a fresh deployment, i.e. exactly when a rollout is being
  watched. Use an exact `COUNT(*)` where the table is bounded by configuration.
- **golang-migrate runs a migration file as a single Exec**, which Postgres treats
  as an implicit transaction, so `CREATE INDEX CONCURRENTLY` is rejected. No
  migration in this repo uses it.
- **`WHERE id IN (...) ORDER BY id FOR UPDATE` does lock in sorted order.** EXPLAIN
  shows `LockRows` above `Sort`. That is what makes the batched write
  deadlock-free, and it is pinned by a test driving two goroutines over the same
  worker set in opposite orders.
- The connection pool already guards the old GOAWAY-4101 relay regression:
  `LimitedReuseMaxStreams` defaults to 0, so relayed ("limited") connections stay
  single-use and reuse only ever applies to directly-dialed workers. Pooling is
  less risky than `grpc-pool-reuse-relay-regression` suggests.
- `hermes chat` still fails silently on vault writes (kiro credentials 404, exit
  code 0) - see `hermes-vault-write-broken`.

## Running the tests

`coord/db` tests skip silently without Postgres:

```
DEPIN_TEST_POSTGRES='postgresql://depin:depin@localhost:15445/depin_test?sslmode=disable' \
  go test ./coord/... ./internal/testplanet/...
```

`scripts/audit-diagnose.sql` is the read-only prod counterpart (EXPLAIN ANALYZE of
both slow statements, index usage, TOAST size, wasted-claim share); run it before
and after migration 000050.

## Deliberately not done

Collapsing the two overlapping `workers` SELECTs per segment
(`GetWorkersByAliases` + `GetWorkersForAudit`). They have different filters and
projections, and merging them means changing `order.Service.CreateAuditOrderLimits`,
which other callers share - a real correctness risk for ~1.5% of the budget.

Related: `audit-subsystem`, `coord-metrics-collection-shipped`,
`grpc-pool-reuse-relay-regression`.
