---
type: fact
tags: [depin, reward, compensation, billing, payments, aioz, fx, coord]
created: 2026-08-07
agent: main
---

Worker payouts converted USD -> AIOZ, plus a client-facing price RPC. Branch
`feat/reward-flows`, uncommitted. Closes the gap left by
[[worker-compensation-phase2]] and [[client-billing-phase-a]]: both halves of the money layer
existed, but `coord/deposit.usdValue` (AIOZ -> USD) had **no inverse anywhere in the repo**, so
the CLI could say a worker was owed $14.32 and nothing could say how many AIOZ that is.

Plan: `/home/tuan/.claude/plans/to-calculate-worker-reward-parallel-balloon.md`. The vault copy
could NOT be written — `hermes chat -m kr/claude-sonnet-4.5-agentic` returns
`HTTP 404: No active credentials for provider: kiro` (9router at 127.0.0.1:20128). Re-auth kiro
and re-run to land it in `Projects/depin/plans/2026-08-07-worker-reward-aioz-conversion.md`.

**User's four decisions** (asked up front with worked examples): volume-weighted average
(`SUM(amount_usd)/SUM(amount_aioz)`), **lifetime** window (no time filter — user overrode my
recommendation of per-period), fall back to live spot when no deposits exist, and freeze the
AIOZ amount at record time on `worker_payments`.

**The load-bearing idea: the atto scales cancel.** USD is attousd (1e-18) and AIOZ is attoaioz
(1e-18), so `attousd / attoaioz = USD per AIOZ` directly. All conversion happens in base units;
there is no scaling step to get wrong and no float near money.

**Shipped**
- `internal/fx` — new leaf below both `internal/billing` and `internal/compensation` (neither
  can import the other; both are leaves). `AiozToUSD`, `USDToAioz`, `WeightedAverageRate`,
  `DeviationPercent`. `coord/deposit.usdValue` + `attoPerAioz` deleted and migrated onto it.
- `billing.DepositTotals` + `BillingRepository.TreasuryTotals` — one unfiltered SUM/COUNT/MIN/MAX
  over `client_deposits`.
- Migration `000044_worker_payments_aioz`: `amount_aioz NUMERIC(60,0)` + `usd_rate NUMERIC(40,18)`,
  same shapes as `client_deposits` so the two halves of the book compare.
- `deposit.ResolvePayoutRate(ctx, TreasuryReader, RateSource)` — treasury first, spot only when
  `Count == 0`. Lives in `coord/deposit`, NOT in `cmd/coord`, specifically so the fallback is
  testable without a CLI or a database.
- CSV columns: paystub gains `payable-aioz` + `aioz-usd-rate`; payment CSV becomes
  `period, worker-id, amount, amount-aioz, usd-rate, to-address, receipt, notes`.
- `compensation.PaymentsFromInvoice` + new `coord compensation payouts-csv <paystubs.csv>` —
  a pure offline transform, no DB and no second conversion, so the AIOZ an operator sends is
  provably the AIOZ that was quoted.
- `generate-invoices-csv` gains `--rate-override` and `--max-rate-deviation-percent`, and prints
  the rate basis (source, deposit count, totals, window) to stderr before writing anything.
- `BillingService.GetAiozPrice` (5th RPC) + `coord/peer.go billingRateSource()`.

**Non-obvious things worth keeping**

- **Two different AIOZ/USD rates now exist and confusing them is a money bug.** Clients are
  quoted **live spot** (that is what credits their deposit at landing); workers are paid at the
  **lifetime volume-weighted** blend. Quoting the payout rate to a client tells them to send the
  wrong amount. Written into the proto comment, `coord/billing/README.md` and
  `coord/deposit/README.md` §4.1 because it is not guessable from the code.
- **`decimal.Div` truncates at `DivisionPrecision = 16`, two short of atto.** $1.00 at $0.03/AIOZ
  comes out `33333333333333333300` instead of `...333` if you convert in whole-AIOZ space and
  scale up. Every division in `internal/fx` names its precision; there is a test whose only job
  is to fail if someone switches one back to `Div`. **The old `coord/deposit.usdValue` had this
  bug** (`Div(attoPerAioz)`); it is gone now.
- **Real bug found by the e2e, not by any unit test**: `Config.Parse()` returns early when the
  watcher is disabled, leaving `Parsed.Rate` at zero — so the **api** role built a `FixedSource`
  of $0.00 and `GetAiozPrice` reported Unavailable forever. Fixed by splitting out
  `Config.ParseRate()` (same pattern as the existing `ParseSweep()`), used by `billingRateSource`
  and the compensation CLI. The watcher runs on **core**, the price RPC on **api**, so
  `deposit.rate-*` now matters on api nodes too.
- **A zero rate is refused inside `fx.USDToAioz`, not at the call site.** Dividing a payout by
  zero mints an unbounded AIOZ payout for every worker in the run. Same instinct as
  [[sdk-owns-safety-defaults]].
- **Dust deposits (`amount_usd` rounded to 0) are deliberately counted** in the denominator. They
  are AIOZ the network holds and did not pay for; excluding them prices payouts above what the
  treasury cost. There is a test asserting free AIOZ halves the blended rate.
- `ReadPaymentCSV` used fixed indices `rec[3..5]` mixed with a sequential `fieldReader`. Adding
  columns mid-list would have shifted `to-address` into `receipt`; every column now goes through
  the reader (added `fieldReader.text()`), so offsets cannot drift again.
- `coord/db/README.md` in this worktree still listed the long-deleted `RewardRepository`; fixed
  in passing.

**Verified**: `internal/fx` (18 subtests incl. the DivisionPrecision regression and the user's
worked example `0.050396039603960396`), `coord/deposit`, `coord/billing`, `coord/db` (Postgres),
`TestCompensationEndToEnd` (extended: two deposits at different rates -> treasury rate ->
persisted `amount_aioz` reconciles back to the USD owed), two new testplanet tests for
`GetAiozPrice` including quote-then-send-then-read-balance. Plus a **real CLI run** against a
throwaway `reward_aioz_check` database: rate `0.050396039603960396` from a seeded 2-deposit
treasury, `$20 -> 396856581532416503259 attoaioz`, `record-period` run twice recorded once, SQL
cross-check `sum(amount_aioz) * usd_rate == sum(amount)` exactly; override, spot fallback, the
hard refusal with no rate, and the deviation guard (74.80% refused at 10%, allowed at 90%) all
confirmed.

**Pre-existing failures, confirmed by stashing everything and re-running**: `TestGetExpired`
(`worker/pieces`) fails on a clean baseline. `internal/testplanet` is flaky under full-suite
load — a *different* test failed on each full run (`TestRepairCloneRebuildsLostReplica`, then
`TestGCReclaimsDeletedPieces`) and both pass 3/3 in isolation. Nothing in the money path.

Local Postgres for tests: `DEPIN_TEST_POSTGRES=postgresql://admin:admin123@localhost:5445/hub?sslmode=disable`
(docker container `coord-db`; there is no host `psql`, use `docker exec -i ... psql`). The
compensation CLI's DSN flag is `--app-config.db.postgres-dsn`, not `--db.postgres-dsn`.
