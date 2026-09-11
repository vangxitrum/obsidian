# Speed up the coordinator audit pipeline (and make it observable)

## Context

The `coord audit` role is auditing ~3,300 segments/hour/process (2 replicas → ~6,500/h),
and its logs are full of GORM `SLOW SQL >= 200ms` warnings from two statements:
`coord/db/reputation_repo.go:120` and `coord/db/audit_repo.go:245`.

Tracing it against the live Prometheus (`monkit` `function_times`, instance
`8353011db829`, 76,893 segments audited) gives the real per-segment budget:

| phase | s/segment | share |
|---|---|---|
| `Verifier.Verify` (network fetch of ~61 pieces) | 1.407 | 58% |
| ↳ of which `Pool.get` / `dialEncryptedConn` | 22.3 CPU-s (368 ms × 61) | |
| `reputation.ApplyAudit` (61 serial RMW transactions) | 0.807 | 33% |
| `CreateAuditOrderLimits` | 0.115 | 5% |
| `GetWorkersForAudit` + bandwidth batch + `Next` + `ContainedWorkers` + `GetAuditSegment` | 0.096 | 4% |
| **total** | **2.43** | |

2.43 s ÷ `WorkerConcurrency=2` = 1.21 s wall/segment ≈ 2,970 seg/h — matching the
observed 3,279. So the slowness has three independent causes, and "Postgres is slow"
is not one of them:

1. **The reputation write is 61 serial transactions per segment.**
   `coord/audit/reporter.go:64` loops one `ApplyAudit` per holder; each is
   `SELECT … FOR UPDATE` → JSON decode → mutate → 16-column `UPDATE` that rewrites the
   ~4 KB TOASTed `audit_history` JSONB (`coord/db/reputation_repo.go:59-155`).
   4.65 M such transactions on one instance. This is the `reputation_repo.go:120`
   slow line, and 33% of the per-segment cost.
2. **Every piece fetch opens a fresh TLS connection.** `coord/peer.go:241` builds
   `dial.NewDefaultDialer` — no pool — so `Pool.get` averages 368 ms and is paid 61×
   per segment (4.65 M dials). `internal/grpcutil/dial/dial.go:69` already provides
   `NewDefaultPooledDialer`.
3. **`WorkerConcurrency` defaults to 2** (`coord/audit/config.go:22`). `Verify`
   highwater is pinned at 2 while `Fetch` highwater reaches 160 — the network layer is
   nowhere near saturated, the worker loop is just throttled.

Secondary: `reverification_queue` has no index supporting
`COALESCE(last_attempt, inserted_at)` (only `inserted_at`, `000024_audit_queues.up.sql:35`),
so `GetNextJob` scans — that is the `audit_repo.go:245` line, fired every 5 s per
process against an *empty* queue (17,036 of 17,204 calls returned `ErrQueueEmpty`).
`ContainedWorkers` filters on `worker_id`, also unindexed. And 17.6% of claimed
segments (16,488 of 93,800) are already deleted by claim time.

Existing metrics (`audit_segments_*`, `audit_verify_duration`) are blind exactly where
the time goes: no queue depth, no phase split, no dial-vs-transfer split, no DB pool
stats. The diagnosis above only worked because `mon.Task()` spans happen to be
exported; that is not a substitute for real instrumentation.

Outcome: cut per-segment cost from ~2.43 s to well under 1 s, raise sustained
throughput several-fold, silence both slow-SQL statements, and ship the audit metrics
+ dashboard that make the next regression self-evident.

---

## Phase 0 — Instrumentation (ships first, so before/after is provable)

Follow the existing monkit idiom exactly: package-level `mon` (already at
`coord/audit/observer.go:21`), `mon.Counter/DurationVal/IntVal`, `monkit.NewSeriesTag`
for dimensions, and the `withTestMon(t)` isolation pattern in
`coord/audit/monkit_test.go:17`.

**`coord/audit/worker.go`** — split `auditOne` into timed phases:
- `audit_phase_duration` (DurationVal, tag `phase=claim|load|verify|report|contain`).
  `verify` replaces nothing — keep `audit_verify_duration` for dashboard continuity.
- `audit_slot_wait_duration` — time blocked on `limiter.Go` (`worker.go:95`), the
  metric that would have shown `WorkerConcurrency=2` was the ceiling.
- `audit_segments_skipped` (Counter, tag `reason=expired|deleted|scheme|no_root|no_merkle|inconclusive`)
  for the early returns at `worker.go:103/107/126`. Today they vanish at Debug level.

**`coord/audit/fetch.go`** — split `Fetch`/`FetchWholePiece`/`FetchChunkProof`/`FetchSignedPiece`:
- `audit_fetch_dial_duration`, `audit_fetch_transfer_duration`, `audit_fetch_bytes`.
  Dial elapsed is currently only a Debug log (`fetch.go:192`). Classify transport
  (direct vs relayed) *after* the dial returns, per the download-tracing precedent.

**`coord/audit/reporter.go`** — `audit_report_duration`, `audit_reputation_updates`
(Counter), `audit_reputation_batch_size` (IntVal).

**New `coord/audit/queuedepth.go`** — a `sync2.Cycle` chore (1 min) emitting
`audit_queue_depth` and `audit_reverify_queue_depth`. Back it with new repo methods
using `pg_class.reltuples` estimates, not `COUNT(*)`, so the gauge never becomes its
own load. Register in `coord/audit_peer.go` next to the workers.

**`coord/db/database.go`** — the whole `coord/db` package has zero DB metrics today
(`coord/db/db_monkit.go` declares `mon` and nothing uses it beyond one domain counter).
Chain `sql.DB.Stats()` into monkit behind a `sync.Once`, mirroring
`chainEndpointStatsOnce` at `worker/peer.go:494` (duplicate chained sources produce
duplicate samples that remote_write rejects): `db_pool_open`, `db_pool_in_use`,
`db_pool_idle`, `db_pool_wait_count`, `db_pool_wait_duration_ns`.

**`coord/db/database.go`** — replace the bare `&gorm.Config{}` at line 20 with an
explicit `logger.New(...)` whose `SlowThreshold` comes from config
(`--db.slow-query-threshold`, default `200ms`). Right now the threshold is gorm's
hardcoded default with no way to tune the log volume.

**`monitoring/grafana/dashboards/coord/coord-audit.json`** — new dashboard, uid
`coord-audit-v1`, tags `[aioz-depin, coord, audit]`, `${DS_PROMETHEUS}` datasource
variable and the `role=~"$role"` multi-select matcher every other dashboard uses. The
`coord/` subdirectory already has a provisioning provider, so no `dashboards.yml`
change is needed. Rows: Throughput (segments/h, phase breakdown stacked), Queues
(depth + claim latency + wasted claims), Fetch (dial vs transfer, p-values), Reputation
(batch size, update rate, duration), DB pool.

Caveat to note in the dashboard description: `pkg/telemetry/config.go:58`
`DropQuantiles` defaults true, so `r99` is not available — panels use
`ravg`/`max`/`sum`-over-`count`.

---

## Phase 1 — Batch the reputation write (removes 33% of per-segment cost)

The single biggest DB win and the direct fix for the `reputation_repo.go:120` slow line.

**`coord/reputation/reputation.go`** (the `DB` interface) — add:

```go
UpdateBatch(ctx context.Context, ids []string, mutate func(Info) Info) (map[string]Info, error)
```

**`coord/db/reputation_repo.go`** — implement it as one transaction with exactly two
statements, modelled on `AccountingRepository.AddWorkerBandwidthAllocatedBatch`
(`coord/db/accounting_repo.go:427-474`), whose doc comment already records that
replacing ~35 sequential round trips was what fixed a 50 s download path:

1. `SELECT <the 16 reputationRow columns> FROM workers WHERE id IN (?) ORDER BY id FOR UPDATE`
   — **`ORDER BY id` is load-bearing**: it makes lock acquisition order deterministic
   across concurrent audit processes, which is what keeps this deadlock-free.
2. Decode / `mutate` / re-encode in Go (unchanged logic).
3. One `UPDATE workers AS w SET … FROM (VALUES …) AS v(id, …) WHERE w.id = v.id`
   for all N rows.

Keep the existing single-row `Update` — `RecordWorker` (the reverify path) still uses it.

**`coord/reputation/service.go`** — add `ApplyAuditBatch(ctx, deltas []Delta, now)`
that reuses the exact same `AddToHistory` → `OnlineScore` → `s.apply` closure as
`ApplyAudit` (`service.go:52-67`), then emits `reputation_worker_disqualified` per
newly-disqualified worker.

**`coord/audit/reporter.go`** — `Record` calls `ApplyAuditBatch` once with the
aggregated deltas instead of looping. This also fixes a latent correctness bug: the
current loop `return`s on the first error (`reporter.go:66-68`), silently dropping the
remaining holders' outcomes.

Do **not** resurrect `AuditRepository.AddAuditCounts` / `addCountsWithHistory`
(`coord/db/audit_repo.go:303-389`). They are dead code and are still row-at-a-time
inside a transaction — the same problem with extra indirection. Delete them in this
phase (with their tests) rather than leaving two competing write paths.

Expected: 0.807 s → ~0.03 s per segment, and 4.65 M `UPDATE workers` statements/hour
collapse to ~76 k.

---

## Phase 2 — Pooled dialer for the audit fetcher (gated, default off)

368 ms of TLS handshake × 61 pieces × every segment. `NewDefaultPooledDialer` already
exists at `internal/grpcutil/dial/dial.go:69`.

This is deliberately gated because reuse over a *limited* relay circuit is known to die
after ~3 pieces (GOAWAY 4101). So:

- Add `Audit.PooledDialer bool` (default `false`) + `Audit.DialPoolCapacity int`
  (default `256`) to `coord/audit/config.go`.
- In `coord/audit_peer.go`, when enabled, build a **dedicated** pooled dialer for the
  audit fetcher rather than swapping `b.Dialer` for the whole peer — blast radius stays
  inside audit. `NewDefaultConnectionPool` (`dial.go:80`) uses `Capacity: 100,
  KeyCapacity: 5`; 61 distinct worker keys × concurrency 8 would thrash it, so size the
  pool from config.
- Reuse the existing no-reuse-over-limited-conn guard from the relay-reuse work; if a
  conn is limited, fall through to a fresh dial.
- `fetch.go` currently calls `conn.Close()` after each fetch (`fetch.go:216`) which,
  pool-less, tears the connection down. With a pool that becomes a release — verify
  this is true for the pooled path and not a silent leak.

Rollout: enable on one of the two audit replicas, compare `audit_fetch_dial_duration`
and `audit_phase_duration{phase="verify"}` in Grafana for 24 h, then flip the default.

---

## Phase 3 — Concurrency and claim batching

Raising concurrency alone would just move the bottleneck onto the serial claim loop
(`worker.go:80` claims one row per round trip, so claim rate == completion rate), so
these two land together.

- **`coord/audit/store.go`** — extend `VerifyQueue` with
  `NextBatch(ctx, n int) ([]QueueSegment, error)`; keep `Next` as `NextBatch(ctx, 1)`.
- **`coord/db/audit_repo.go`** — one `DELETE … WHERE segment_id IN (SELECT … ORDER BY
  inserted_at FOR UPDATE SKIP LOCKED LIMIT n) RETURNING …`, same shape as the existing
  `Next` (`audit_repo.go:177-186`), which stays correct for multi-process claiming.
- **`coord/audit/worker.go`** — claim `WorkerConcurrency` items per round trip and feed
  them to the limiter.
- **`coord/audit/config.go`** — `WorkerConcurrency` `2 → 8`, `ReverifyConcurrency`
  `2 → 4`. These are per-process and the fleet runs 2 replicas.

Also add a `Next`-side join against `segments` so ids deleted since push are dropped in
the same statement instead of being claimed and then failing `GetAuditSegment` — 17.6%
of claims today. Count what is dropped via `audit_segments_skipped{reason="deleted"}`.

---

## Phase 4 — The two slow statements, directly

**New migration `coord/db/migrations/000044_audit_queue_indexes.up.sql`** (confirm the
next free number at implementation time — a duplicate migration number has broken this
repo before):

```sql
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_reverification_queue_next_attempt
    ON reverification_queue ((COALESCE(last_attempt, inserted_at)));

CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_reverification_queue_worker_id
    ON reverification_queue (worker_id);
```

The first makes `GetNextJob`'s predicate indexable; the second serves
`ContainedWorkers` (`audit_repo.go:273`), which runs once per segment. Note
`CONCURRENTLY` cannot run inside a transaction — check how golang-migrate is configured
here and split the file or set `-x` accordingly.

**`coord/db/audit_repo.go:233-245`** — change `ORDER BY inserted_at` to
`ORDER BY COALESCE(last_attempt, inserted_at)` so the plan is a pure index scan.
Semantics are unchanged (still oldest-attempt-first).

**`coord/audit/reverifyworker.go:100-105`** — exponential backoff on `ErrQueueEmpty`
(5 s → 60 s cap, reset on any hit). Today an empty queue costs one scanning `UPDATE`
every 5 s per process, forever, which is most of the slow-SQL noise.

**`coord/db/worker_repo.go:85-106`** — `GetWorkersByAliases` uses GORM `Find` with no
`Select`, so it emits `SELECT *` and drags `audit_history` (~4 KB, TOASTed) plus
`egress_nets` off disk for all ~61 holders on every segment, even though
`contact.Worker` doesn't map `audit_history` at all. Add an explicit narrow column
list. Same for `GetWorkersForAudit` (`worker_repo.go:114-150`), which lists columns
explicitly but still includes `audit_history`.

Also worth collapsing: the verifier issues **two** overlapping `workers` selects per
segment — `GetWorkersByAliases` (`verifier.go:314`) and then `GetWorkersForAudit` via
`CreateAuditOrderLimits` (`verifier.go:320` → `order/service.go:410`). Pass the already-
resolved set through instead of re-querying. Worth ~37 ms/segment and one round trip.

---

## Files touched

| Area | Files |
|---|---|
| Metrics | `coord/audit/{worker,fetch,reporter}.go`, new `coord/audit/queuedepth.go`, `coord/audit_peer.go`, `coord/db/database.go` |
| Reputation batching | `coord/reputation/{reputation,service}.go`, `coord/db/reputation_repo.go`, `coord/audit/reporter.go` |
| Dialer | `coord/audit/config.go`, `coord/audit_peer.go`, `coord/audit/fetch.go` |
| Queue / concurrency | `coord/audit/{config,store,worker,reverifyworker}.go`, `coord/db/audit_repo.go` |
| Schema | `coord/db/migrations/000044_audit_queue_indexes.{up,down}.sql` |
| Query shape | `coord/db/worker_repo.go`, `coord/order/service.go`, `coord/audit/verifier.go` |
| Dashboards | `monitoring/grafana/dashboards/coord/coord-audit.json` |
| Diagnostics | new `scripts/audit-diagnose.sql` |

---

## Verification

**Unit / package.** Every touched package gets tests in its existing idiom:
`withTestMon(t)` (`coord/audit/monkit_test.go:17`) for the new counters; a real
Postgres-backed test for `UpdateBatch` asserting (a) N workers updated in one
statement, (b) `audit_history` windows fold identically to the current per-row path
for the same input, (c) two concurrent batches with overlapping worker sets do not
deadlock. Run `go build ./...` after each phase — the LSP is unreliable in this repo.

**End-to-end.** Extend the testplanet audit test to run a segment with multiple holders
and assert one batched `UPDATE` (via the new `audit_reputation_batch_size` metric) and
correct final reputation. Full `go test ./coord/...` must stay green.

**SQL plans (you run this).** Ship `scripts/audit-diagnose.sql` for the prod coordinator
DB — read-only, safe to run live:
- `EXPLAIN (ANALYZE, BUFFERS)` on both slow statements, before and after the migration.
- `pg_stat_user_indexes` for `reverification_queue` / `audit_queue` / `workers`, to
  confirm the new indexes are actually chosen.
- `pg_total_relation_size` vs `pg_relation_size` on `workers` (TOAST bloat from
  `audit_history` rewrites) and `n_dead_tup` / `last_autovacuum` from
  `pg_stat_user_tables`.
- row counts of `audit_queue` and `reverification_queue`, to sanity-check the new depth
  gauges.

**Live before/after, in Grafana.** The Phase 0 metrics are the acceptance test.
Targets after all phases, on the same fleet:
- `audit_phase_duration{phase="report"}` ravg: 0.807 s → < 0.05 s
- `audit_phase_duration{phase="verify"}` ravg: 1.41 s → < 0.9 s (Phase 2 enabled)
- segments/h/process: ~3,300 → > 10,000
- `SLOW SQL >= 200ms` occurrences for `reputation_repo.go` and `audit_repo.go`: → ~0
- `audit_queue_depth` trending flat or down, not accumulating

**Rollout order.** Phase 0 alone first, one release, so the baseline is recorded by the
new metrics. Then 1 + 3 + 4 together. Then Phase 2 on one replica only, compared
against the other, before changing its default.

---

## As built (2026-08-26)

Deviations from the plan above, all of them found during execution:

- **Migration number is 000050**, not 000044 (that slot was already taken), and the
  indexes are **not** `CONCURRENTLY`: golang-migrate executes a migration file as a
  single Exec, which Postgres runs as an implicit transaction, and
  `CREATE INDEX CONCURRENTLY` is rejected inside one. `reverification_queue` holds
  hundreds of rows, so a plain build is instantaneous.
- **Queue depth uses an exact `COUNT(*)`, not `pg_class.reltuples`.** The estimate
  is -1 on a never-analyzed table, so the gauge would read 0 on a fresh deployment -
  exactly when a rollout is being watched. Both queues are bounded by the audit
  config rather than by data volume, so the count is cheap. The e2e test caught this:
  it passed vacuously against the estimate.
- **`GetWorkersForAudit` was left alone.** The plan suggested dropping
  `audit_history` from its projection, but `projectWorker` decodes it to derive the
  online score that `ReputationFilter.MinOnlineScore` reads. Only
  `GetWorkersByAliases` was narrowed, via gorm's own schema parse so the list cannot
  drift from `contact.Worker`.
- **Collapsing the two overlapping `workers` SELECTs per segment was not done.**
  They have different filters and projections, and merging them means changing
  `order.Service.CreateAuditOrderLimits`, which other callers share. That is a real
  correctness risk for ~1.5% of the per-segment budget.
- Added beyond the plan: a `--app-config.db.slow-query-threshold` flag (the 200ms
  was gorm's hardcoded default with no override), and `scripts/audit-diagnose.sql`
  validated end to end against a live migrated database.

Two real bugs were caught by the new tests before shipping:

1. **gorm silently scans nothing into an embedded struct.** The first `UpdateBatch`
   used `struct{ ID string; reputationRow }`; gorm zeroed the embedded row, so the
   mutator got a blank `Info` and the absolute UPDATE **reset every worker's
   reputation**. Fixed by flattening the id into `reputationRow`.
2. The `reltuples` gauge above.

Lock ordering was verified rather than assumed: `EXPLAIN` on
`WHERE id IN (...) ORDER BY id FOR UPDATE` puts `LockRows` above `Sort`, so rows
really are locked in id order, and a test drives two goroutines over the same worker
set in opposite orders to pin it.
