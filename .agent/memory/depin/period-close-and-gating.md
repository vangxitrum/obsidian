---
type: fact
tags: [depin, billing, compensation, period-close, gating, coord, money]
created: 2026-08-17
agent: main
---

Automatic monthly period close, an intra-period accrual watchdog, and an upload/download
payment gate. Branch `fix/payment`, uncommitted. Closes the last gap left by
[[client-billing-phase-a]] and [[worker-payout-aioz-conversion]]: both ledgers existed and
both were correct, but nothing ran them, and nothing stopped an unfunded client using the
network. Plan: `/home/tuan/.claude/plans/now-i-want-it-mellow-cook.md` (the vault copy
failed again — hermes still returns `HTTP 404: No active credentials for provider: kiro`).

**Shipped**: `coord/periodclose` (shared pricing extracted out of `cmd/coord`, the close
`Service`, the `Watchdog`, an `Accrual` reader), `coord/billing.Gate`, migration
`000047_period_close`, `coord/db/periodclose_repo.go`, `coord period-close status`,
`vo.PaymentRequired` (gRPC FailedPrecondition / HTTP 402), `GetBalance` gains
`accrued_unbilled`/`suspended`/`suspension_reason`, and a `Coord Billing` Grafana dashboard.

**The user's decisions**, asked up front and then again when they spotted the hole
themselves ("what if their balance can not affort the next month cost"): both ledgers,
price *and* record, monthly after a grace delay, **on by default**; then charge in full and
allow a negative balance, **include** gating for uploads *and* downloads, and suspend at a
threshold rather than only alerting.

## What is worth remembering

- **On-by-default is only defensible because of the floor.** `period_close_floor` is a
  one-row table established on the chore's first pass, which closes nothing, and never
  moved by configuration afterwards. Without it an upgraded coordinator carrying a year of
  tallies would find twelve unclosed periods and debit every client for all of them on its
  first tick. Established with INSERT..ON CONFLICT DO NOTHING then read back, so two
  coordinators started in the same second adopt one floor rather than two.
- **Money in before money out.** Client half first: accounts debited for a month the
  network has not yet committed to paying for is a survivable interruption, because nothing
  can be withdrawn against a paystub that does not exist. The reverse lets workers withdraw
  against a debt nobody was billed for. Same reasoning makes a margin failure block the
  whole period rather than just the client half.
- **Oldest-first and serial is load-bearing**, not tidiness: `GenerateStatements` releases
  withheld bond from the lifetime paystub balance, so closing a later period over a failed
  earlier one computes the release against a ledger with a hole in it. A failure stops the
  sweep.
- **The close writes no AIOZ rate.** Paystubs only; the conversion happens when a transfer
  is actually sent. Migration 000045 already argued that an early rate is a free option for
  whoever picks the moment, and a monthly close widens that window to a month. The payoff is
  that month end has no price-API dependency at all.
- **`period_closures` is a journal, not a lock.** The ledgers are authoritative and already
  idempotent, so a lost journal row costs one wasted re-close. A failed close writes
  nothing — a missing row reads as "not closed", which is the safe reading.
- **The gate caches only suspended accounts**, so a healthy network pays one empty-map
  lookup per request, and it **fails open**. A billing outage must not stop the network
  serving data; that risk is exactly why Phase A shipped ungated.
- **`buildManifest` is the only correct download hook.** The coordinator-minted HMAC ticket
  is a pure bearer capability that derives no owner at all, and the client-minted one is
  verified offline, so a check at any individual RPC leaves a way around it.

## Two real bugs found

1. `CompensationRepository.RecordPeriod` zeroed `worker_paystubs.distributed` on every
   re-close (`PaystubFromStatement` never sets it, and the upsert was `UpdateAll: true`).
   Inert today, but `withdrawal_repo.go` sums that column into what a worker may withdraw.
   Fixed with an explicit update-column list, and the regression test was confirmed to fail
   with `UpdateAll` restored before being accepted.
2. The client charge is **frozen at the first close** — the uniqueness index is
   `(owner_id, period)` with no amount in it, so re-closing restates the invoice and leaves
   the original charge standing. A close that runs early under-bills permanently. That is
   why `GraceDelay` defaults to 72h.

## Traps for next time

- `ResourceExhausted` is already load shedding and the edge maps it to a 503 with
  Retry-After, so reusing it for suspension would have clients retrying a wall forever.
  `PermissionDenied` reads as permanent. 402 is the only one a client can act on.
- **testplanet has already run the chore** before a test body starts — the cycle fires
  immediately at boot — so a floor exists. Tests need to clear or move it; `EstablishFloor`
  alone is a no-op by design. This cost one confusing failure.
- `GetDownloadTicketRequest` requires `expires_at`.
- Watchdog interval is 1h because the inputs only change hourly, not out of caution;
  polling faster returns byte-identical numbers at full cost.
- `vo.USD` has `Cmp`/`Neg`/`IsNegative` but no `Equal`. `AiozCoin.AiozAmount()` is in atto
  units. Metrics carry money in cents because attousd overflows int64.
- Test Postgres in the treehouse worktree is
  `postgresql://depin:depin@localhost:15445/depin_test` (container `depin-pgtest`), not the
  `coord-db`/5445 DSN older notes give.

**Verified**: 41 unit tests in `coord/periodclose`, 9 Postgres tests, the gate tests, the
`vo`/edgeserver 402 mapping, 5 testplanet e2e including a three-way idempotency proof (run
again, then `CloseOne` bypassing the journal) and the bearer-ticket refusal, plus a real CLI
run of `coord period-close status` against a freshly migrated throwaway database. Two
consecutive clean full testplanet runs; one earlier run failed
`TestDownloadFailsWhenMidStreamLossIsUnrecoverable`, which passes 3/3 in isolation — the
known full-suite flakiness, not this work.
