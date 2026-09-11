# Billing / settlement audit against live hub DB (2026-08-20)

DB audited: `postgresql://admin:admin123@68.183.189.51:5445/hub` (live coordinator).
No psql on this box - used `docker run --rm -e PGPASSWORD=... postgres:16-alpine psql -h ... `.
Prometheus for the same fleet is local at `127.0.0.1:9090` (monitoring/ stack) and was
decisive for separating "settlement rejected" from "settlement never attempted".

## Finding 1 (severity: revenue) - worker order settlement DESTROYS windows on any transient trust error

FIXED 2026-08-20 in `worker/orders/service.go` (uncommitted, needs a worker rollout).

**First read of the data was wrong and cost time - record it so nobody repeats it.** The DB
shape (`allocated` growing to today, last `settled > 0` bucket at 2026-08-14) looks exactly
like "settlement is dead". It is not. Two things hid the truth:

1. `OrderLimitGracePeriod` defaults to **48h**, and `ListUnsentByCoord` skips a window until
   `now > CreatedAtHour + 1h + grace`. Settlement therefore lags **~49h by design**. The last
   settled bucket is always about `now - 49h`, which reads as "stopped 2 days ago" on every
   healthy day. Never compare `settled` to `allocated` inside a 49h window.
2. `order_settlement_rows` is created lazily inside the success branch, so the Prometheus
   series is ABSENT (not zero) whenever a coord process has settled nothing yet. Absent series
   != no traffic.

What proved it: `sum(settled)` moved 35.90 GB -> 38.50 GB in 40 minutes while `allocated` did
not move at all, and the bucket that filled was exactly `now-49h` (2026-08-18 09:00).

**The real defect** is in `SendOrders`: the guard after `trustSource.GetNodeURL` was commented
out, so EVERY resolve error - not just `trust.ErrUntrusted` - set `skipSettlement = true` and
then fell through to `ordersStore.Archive(...)`. An archived window is never resent, so one
transient error permanently destroys every order in that hour, for every action at once
(one orders file per (coord, hour) holds PUT/GET/GET_AUDIT/GET_REPAIR/PUT_REPAIR together).
That is why all five actions die on the same timestamp - no download-path theory can explain
that, and it is the tell that separates this from a traffic or SDK problem.

Confirmed lost in production: every 2026-08-17 bucket (~68 GB GET allocated) became sendable
on 2026-08-19 and is still exactly 0 after many hourly sender ticks, while 2026-08-18 09:00
settled normally the moment it became eligible. Cliffs line up with deploy days (08-14, 08-17).

Fix = restore Storj's rule (`if !errs.Is(err, trust.ErrUntrusted) { addErrorCoord; return }`)
plus `orders_send_deferred` / `orders_dropped_untrusted_coord` / `orders_dropped_bytes` metrics
and the dropped byte count in the warn log, because nothing previously distinguished lag from
loss. Regression tests: `worker/orders/service_test.go`
(`TestSendOrders_TransientTrustFailureKeepsOrdersUnsent` reproduces with the real production
error string `"coord db error: context canceled"`; `..._UntrustedCoordArchivesOrders` guards
the other half). Note `internal/testplanet` TestWorkerBandwidthRollupAllocatedVsSettled PASSES
against the broken code - the bug needs a trust failure, which testplanet never has.

Stale trap found nearby: `worker/orders/store_test.go` skips `TestOrdersStore_ListUnsentByCoord`
claiming `ordersfile` filenames cannot round-trip because UUIDs contain hyphens. `SplitChar` is
`":"`, not `"-"` - the comment is wrong and the skip should be removed.

## Finding 2 - client egress billed at manifest issue  [IMPLEMENTED 2026-08-21, BLOCKED on a go-sdk release]

Storj answer, confirmed in source before building: the client's bill comes out of the WORKER's
settled orders. `UpdateBucketBandwidthSettle` runs inside `SettlementWithWindow`, and the
invoice (`GetProjectTotalByPlacement`) sums `settled + inline` from bucket_bandwidth_rollups.
Allocated survives only as the live enforcement estimate, and even there
`GetProjectBandwidth` switches to settled after `allocatedExpirationInDays = 2` via
`CASE WHEN interval_day < ? THEN egress_settled ELSE egress_allocated-egress_dead END`.

Ported: migration 000049 renames `egress_bytes` -> `egress_allocated` and adds
`egress_settled` / `egress_dead` / `egress_inline`; existing rows backfill settled = allocated
(no truer number exists for them, and leaving zero would restate every closed period's egress
to nothing). Invoice reads settled + inline.

Attribution rides the limit, Storj-style: `shared.OrderLimit` gains
`encrypted_metadata_key_id` + `encrypted_metadata`, sealed with a new
`coord/order.EncryptionKey` (secretbox, serial as nonce), carrying a new
`hub.orders.v1.OrderLimitMetadata{contract_id}`. Settlement decrypts and credits the contract
INSIDE the same transaction and the same replay guard as the worker credit.

Load-bearing details:
- **The metadata MUST be signed.** `EncodeOrderLimit` builds a separate `OrderLimitSigning`
  message field by field, so new fields are invisible to the signature unless added there too
  (Storj's OrderLimitSigning carries them). Unsigned, a worker could swap metadata between
  limits and move its bandwidth onto another customer's invoice.
- **allocated must be Σ limit.Limit, not the segment's EncryptedSize.** Settled is Σ
  order.Amount over the same limits, so mixing units makes settled read larger than allocated
  (first test failure: 141312 > 70144). Storj sums orderLimit.Limit too, which also means a
  customer pays for bytes the network moved (over-fetch included), not object size.
- **Inline segments need their own column.** No worker, no order, nothing to settle -- billed
  at manifest time and summed with settled at invoice time, exactly as Storj's `inline` is.
  Without it inline downloads are free.
- No config: the key is derived by HKDF from the coordinator's identity PRIVATE key when
  `order.encryption-keys` is unset. NOT a random key -- a random one dies with the process and
  every limit in flight at a restart settles unattributable, opening a ~49h billing hole.

**UNBLOCKED 2026-08-21: go-sdk 73f0c70 shipped**, depin re-pinned to
`v0.0.0-20260821093526-73f0c7097e45`. Original blocker, kept because the failure mode is
non-obvious: The SDK unmarshals the manifest into its OWN generated
`OrderLimit` and re-marshals when talking to the worker; gogofaster drops unknown fields, so a
stale SDK silently strips the metadata and the worker then rejects every limit as badly signed
(`signature is not valid`). Changed in go-sdk (commit 73f0c70): orders.pb.go copied from
depin with `aioz-depin/` -> `aioz-depin/go-sdk/`, OrderLimitSigning spliced into piece.pb.go,
and pkg/signing/encode.go matched.

Re-pin gotcha: `go get gitlab.internal/aioz-depin/go-sdk@<sha>` FAILS ("module declares its
path as: aioz-depin/go-sdk but was required as: gitlab.internal/..."), because the module's
declared path is not its host path -- which is the whole reason the replace directive exists.
It still prints the resolved pseudo-version, so use it to read the version and then edit the
replace line by hand.
- Trap: do NOT copy depin's piece.pb.go wholesale into the SDK. depin's is generated from a
  proto that also defines workerinfo enums, so it panics with
  `duplicate enum registered: worker.workerinfo.v1.FormatVersion`. Splice only the
  OrderLimitSigning declarations.
- Trap: `hack/sync-from-depin.sh` in go-sdk is DISABLED and would `rm -rf` the module.

## Finding 3 - GET_AUDIT never records allocated  [FIXED 2026-08-20]

`coord/audit/verifier.go` and `coord/segmentverify/verify.go` minted GET_AUDIT limits without
the `order.NewAllocation()` + `Record` step every other issuing path performs, so all 15,800
action=3 rows had `allocated = 0, settled > 0` - the exact shape
`coord/order/allocation.go` documents as impossible. Not a money bug (payout sums `settled`,
and `ListWorkerBandwidth` excludes GET_AUDIT outright), but it disarms "settled > allocated"
as a corruption alarm for the WHOLE table, which is why it was worth fixing.

Fix mirrors `coord/repair/repairer`'s `recordAllocation`: a nil-tolerant
`order.AllocationStore` field plus a log-and-swallow helper (audit protects durability;
refusing to audit over an accounting row would trade data safety for bookkeeping).
Instrumented at all 5 mint sites - 4 in coord/audit (2 normal-audit, 2 reverify) and 1 in
coord/segmentverify.

**Load-bearing detail:** record AFTER `selectAuditTargets` trims, not at the mint. The
verifier drops contained/offline holders, and a dropped limit never reaches a worker and can
never settle - counting it would inflate allocated with bandwidth nobody was authorized to
serve. Same reasoning puts segmentverify's call after its retry block.

Wiring touched 4 call sites: `coord/peer.go`, `internal/testplanet/helpers.go` (both the audit
verifier and SegmentVerifyService), and `cmd/segment-verify/main.go`.
Tests: `internal/testplanet/audit_bandwidth_test.go` (both reproduce as `0 is not greater
than 0` pre-fix). They read the table directly because `ListWorkerBandwidth` filters GET_AUDIT
out - the very rows under test.

## Finding 4 - `dead` cannot detect an unsettled order  [FIXED 2026-08-20]

`coord/order/settlement.go` computes `dead += limit.Limit - order.Amount` only for orders the
worker DID settle, so an order that never arrives contributes nothing. Lifetime numbers:
allocated 795.2 GB, settled 35.9 GB, dead 390.9 GB, **368.4 GB unaccounted** - and during the
Finding-1 loss `dead` read 0. Every per-row invariant the schema can express still held.

**Checked Storj before touching it:** `satellite/orders/endpoint.go:388` uses the identical
formula, and Storj puts Dead on the BUCKET rollup only - `StoragenodeBandwidthRollup` has no
Dead field at all. So redefining depin's `dead` would diverge. The gap is a question `dead`
was never shaped to answer, not a bug in how it is computed. Fix = add a signal alongside it.

Added `coord/accounting/reconcile.go`: a `Reconciler` chore that reports, per action,
`allocated - settled - dead` over windows whose settlement deadline has passed. Metrics
`bandwidth_unsettled_bytes`, `bandwidth_unsettled_ratio`, `bandwidth_past_deadline_*`,
`bandwidth_unsettled_total_bytes`, plus a warn log. Backed by
`AccountingRepository.SumUnsettledAllocation(ctx, before)`.

Design decisions worth keeping:
- `SettlementDeadline` default **72h**, NOT zero. The true floor is ~49h (worker
  OrderLimitGracePeriod 48h + the +1h window rule); set it lower and the chore reports the
  entire healthy pipeline as loss every hour and gets muted. See Finding 1.
- Floor at zero **per row inside the SQL** (`SUM(GREATEST(allocated-settled-dead,0))`), not
  after the SUM - otherwise a few rows where settled+dead exceeds allocated net off real loss
  elsewhere.
- Alert on the **ratio**, not the byte count: absolute unsettled tracks traffic volume, so a
  threshold either fires on a busy day or misses an outage on a quiet one.
- Per-action, not one total: a broken pipeline drops every action at once (one orders file per
  coord-hour holds them all), so a single action drifting alone points elsewhere.
- Do NOT add the method to `accounting.Store` - that fat interface has many fakes and adding
  one method broke ~12 test files. Used a narrow consumer-side `UnsettledStore` instead,
  matching `coord/billing.SuspensionStore`'s idiom.
- **No config at all** (user's final call, after a detour through the audit role). Rides the
  existing hourly tally cycle in `setupAccountingTally`; `settlementDeadline` is a const
  (72h) in reconcile.go, not a flag. Rationale kept in the code: the deadline is a
  correctness bound derived from the worker's own OrderLimitGracePeriod default, not an
  operator preference, and a check an operator must switch on is a check that is off when
  it is needed.

Tests: `internal/testplanet/bandwidth_reconcile_test.go` - one proves unsettled == allocated
before settlement and drops after, the other proves fresh rows inside the deadline report zero.

## Finding 5 - no cross-window order replay protection  [FIXED 2026-08-21]

`SettlementWithWindow` deduped serials with a map built fresh per RPC, so it only caught a
serial repeated inside ONE stream. Every check in `verifyOrder` is stateless, so an archived
window streamed back scored full marks and was credited again; payout sums `settled`, and the
extra bytes land in the ORIGINAL hour's row (interval_start = limit.OrderCreation) where
nobody watching today's numbers would see them. Reproduced e2e: settled went 282,624 ->
565,248 by replaying one worker's archived orders over real mTLS as that worker.

Ported Storj's `UpdateStoragenodeBandwidthSettleWithWindow` / `alreadyProcessed`:

- **Storj's marker does not port.** Storj CREATEs storagenode rollup rows at settlement, so
  "rows exist for (node, window)" IS the already-processed test. depin cannot reuse it: the
  ALLOCATED side writes those rows at order-issue time, so a row is always there. Hence an
  explicit journal, `worker_settlement_windows` (migration 000048), PK
  (worker_id, window_start, action).
- The PK does double duty - it is also the concurrency backstop, which is what Storj gets
  from its rollup PK + retry-on-constraint-error. Journal INSERT happens BEFORE the rollup
  credit inside the transaction, so a racing stream rolls back without touching rollups.
- Same amounts resubmitted -> SUCCESS, credit nothing (an honest worker that never saw our
  response must be told to stop retrying). Different amounts -> `ErrWindowAmountMismatch` ->
  REJECTED + `order_settlement_window_mismatch` counter. Compares settled only, not dead
  (dead is derived from the same orders, so it cannot differ alone).
- **One call = one window**, taken from the first order, cross-window orders refused - Storj's
  `isValid` window check. Load-bearing: without it "has this window been credited" has no
  answer, and a worker could dodge the guard by pairing a replayed hour with a fresh one.
- Gotcha: `window` is a RESERVED word in Postgres; column is `window_start`.
- Gotcha: `coord/order` cannot import `coord/accounting` (accounting -> file -> order cycle),
  so `SettledAmount` + `ErrWindowAmountMismatch` live in `coord/order/allocation.go`.
- Endpoint failure semantics changed for the better: the window is now one atomic write, so a
  store error credits nothing rather than partially. A partial credit would make the replay
  guard read it as "already processed" and refuse the retry that would have completed it.

Still absent vs Storj: `verifyOrder` does not check `OrderExpiration`. Storj rejects expired
orders before signature verification.

## Finding 6 - 2026-08 charged mid-period, and is now frozen  [GUARDED 2026-08-21; money not recovered]

Cause and damage unchanged (see below). Two guards added, plus the invariant the user spotted
that ties them together:

- **`GraceDelay` must be >= the settlement horizon**, enforced in `periodclose.Config.Parse()`.
  The worker half is priced on SETTLED bandwidth, and a worker cannot even offer an hour-window
  until ~49h after it opens (its OrderLimitGracePeriod 48h + the hour the window stays open).
  Close sooner and the month is priced against orders that had not arrived - which looks like a
  quiet month, not a bug. Default 72h passes; the bound is `>=`, so equal is legal.
- **One exported constant**, `accounting.SettlementHorizon` (72h), now feeds BOTH the reconciler
  (Finding 4) and this validation. They were two independent numbers that happened to agree;
  now they cannot drift.
- **Manual commands refuse a period that is not ready, `-f/--force` overrides** - one shared
  flag var on both `coord billing record-period` and `coord compensation record-period`
  (`cmd/coord/period_ready.go`). The two halves refuse for DIFFERENT reasons, which is the
  thing to remember: the client half only needs the period to have ENDED (invoices come from
  storage tallies + egress rollups, neither of which waits on a worker), while the worker half
  needs it to have SETTLED. `--force` warns rather than hiding the reason.
- `checkPeriodSettled` separates "too early" (the clock - wait) from "unsettled bytes remain"
  (past the horizon, so it was lost - waiting will not help), because the remedies differ.

NOT done: recovering the ~08-20..08-31 revenue. `idx_client_ledger_charge_once` means the
September close restates the invoice and debits nothing, so `client_invoices.total` will
permanently disagree with `client_ledger` for 2026-08. Options put to the user (detect+report
and fix by hand with `coord billing credit` / auto true-up on re-close / leave it) - undecided.

### Original Finding 6 detail

`client_invoices` + `client_ledger` charge rows for period 2026-08 exist with
`created_at = 2026-08-20 09:37`, while `period_closures` is EMPTY -> this was the manual
`coord billing generate-invoices-csv 2026-08` + `record-period` CLI, not the auto closer
(auto writes `period_closures`). `GraceDelay` is 72h so the closer would not have touched
2026-08 until 2026-09-04.

Consequence is permanent: `BillingRepository.RecordPeriod` deliberately ignores the
"applied" flag and `idx_client_ledger_charge_once` is a unique partial index on
`(owner_id, period) where kind='charge'`. Re-closing 2026-08 in September will restate
the invoice but debit nothing, so 2026-08-20..31 is unbillable. `cmd/coord/billing.go`
has no guard refusing an in-progress period - add one.

Side effect: owner `4f7e6a92-...` now has `balance = -1.376e18` attousd with
`total_deposited = 0`, and `delinquent_since`/`suspended_at` are both NULL because the
accrual watchdog, not the charge, is what stamps them.

## Clean (checked, no defect)

- invoice identity `total = cost_storage + cost_segments + cost_egress - egress_discount`: 0 violations.
- invoice `usage_egress_bytes` == sum of `contract_egress_rollups` for the period: exact for all 20 contracts.
- `client_balances.balance == total_deposited - total_charged`: holds for both accounts.
- ledger charge == sum of that owner's invoices for the period: exact.
- No orphan egress/tally/invoice rows, no rollups for unknown workers, no tallies for DQ'd workers.
- `contract_storage_tallies` totals equal `sum(segments.encrypted_size)` exactly.
- 2026-07 all-zero invoices are correct: earliest file is 2026-08-06.
- Payout half never ran at all: `worker_paystubs`, `worker_payments`, `worker_withdrawals`,
  `period_closures` are all empty. Do not run it until Finding 1 is fixed.

See [[period-close-gating]], [[worker-metering-fix]], [[client-billing-phase-a]].
