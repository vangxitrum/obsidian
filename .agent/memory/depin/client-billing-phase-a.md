---
type: fact
tags: [depin, billing, client-usage, payments, blockchain, aioz, coord]
created: 2026-07-30
agent: main
---

Client usage billing Phase A, built on branch `feat/worker-reward` (uncommitted). The
customer-facing mirror of [[worker-compensation-phase2]]: workers get paid, clients now get
charged. Plan in the vault at
`Projects/depin/plans/2026-07-30-client-usage-billing-onchain-deposits.md`.

**Model** (all four decided by the user up front): prepaid balance debited by period charges;
prices in USD (`vo.USD`, attousd) with AIOZ converted at a rate recorded per deposit;
per-client **custodial** deposit address, reusing the half-built `coord/wallet` + `wallets`
table; Phase A meters/prices/charges but deliberately does **not** gate uploads or downloads.

**Shipped**: `accounting.QueryContractPeriodUsage` (half-open, byte-hours + segment-hours +
egress + owner); `internal/period` extracted so both ledgers agree on what a period covers;
`internal/billing` leaf package; migration `000036_client_billing`
(`client_invoices`/`client_ledger`/`client_balances`/`client_deposits`/`chain_watermarks`);
`coord/deposit` CometBFT block watcher (gated off, core role); `BillingService` RPCs; and
`coord billing generate-invoices-csv|record-period|credit|balances`.

**Traps worth remembering**

- `wallets.address` persists as **hex** (`vo.AccAddress.String()` is EVM-style) while chain
  `transfer` events report **bech32**. Matching them as raw strings credits nobody and looks
  exactly like "no deposits yet".
- Two account keys: `clients.id` (the mTLS peer identity) owns contracts and keys the ledger;
  `clients.address` is what `wallets.owner` references. Everything has to bridge through the
  `clients` row.
- `coord/db` rewrites `gorm.ErrRecordNotFound` into `vo.ErrRecordNotFound` via a query callback,
  so `errors.Is(err, gorm.ErrRecordNotFound)` silently never matches in a repository.
- The client-vs-worker margin check cannot live in either pricing package (both are leaves);
  it lives in `cmd/coord/billing.go`, the one place both configs are bound.
- `AddContractEgress` floors `interval_start` to the top of the hour, so a test window anchored
  at the event time misses the row and egress reads zero. Same trap as [[contract-file-tags-usage]].
- Baseline-comparison gotcha: `git stash` including `go.mod` breaks `internal/testplanet`, because
  go.mod carries the local `replace ../go-sdk`. Stash only the paths you changed.

## EVM switch + sweeper (2026-07-31)

Deposit reading moved to **go-ethereum**, and sweeping to a root wallet was added, at the user's
request.

**Measured on AIOZ mainnet first**: 6/6 blocks that CometBFT reported as carrying transactions
showed **zero** transactions over the EVM JSON-RPC (they were `MsgDelegate` /
`MsgWithdrawDelegatorReward` / `MsgUpdateClient`, all emitting bank `transfer` events), and there
were **no EVM transactions at all** in the last 4000 blocks. So the two chain views are not
interchangeable: Cosmos-native txs are invisible to the EVM RPC, and today that is where all the
traffic is.

I recommended keeping the CometBFT reader; the user chose **EVM-only** with the tradeoff stated,
so a Cosmos-side bank send now lands in the custodial address and is never credited. That is
documented in `coord/deposit`'s package doc, `EVMChain`'s doc and the billing README — clients
must be told to deposit from an EVM wallet.

**Sweeper**: decrypts the custodial ethsecp256k1 key, signs an EIP-155 transfer, sends
`balance - gasLimit*gasPrice` to the configured root wallet (the address pays its own fee, so an
address below the fee is skipped, not failed). `coord deposit sweep` is a **dry run unless
`--broadcast`**. Sweeps are recorded in `wallet_sweeps` (migration 000037) and deliberately never
touch `client_balances` — the client was credited at deposit time, so debiting at sweep time
would double-count.

**Gotchas**: cfgstruct turns `EVMRPC` into `--deposit.evmrpc` unless given `flagname:"evm-rpc"`;
go-ethereum v1.10.26's `types.TrieHasher.Update` returns nothing, so a stub returning `error`
does not satisfy it; a reverted EVM tx is still in the block, so the receipt status must be
checked before crediting.

## Live AIOZ/USD rate (2026-07-31)

The fixed `aioz-usd-rate` config was replaced by a price API (`coord/deposit/rate.go`):
`RateSource` interface, CoinGecko free key-less endpoint by default, `FixedSource` kept for dev
and emergency override, both wrapped in a `CachingSource` that refreshes on an interval and
applies the guards.

**`include_last_updated_at` matters**: an API that answers but serves hours-old quotes is
invisible if you stamp the fetch time instead of trusting the source's timestamp.

Guards, because a bad rate is a silent mis-credit rather than an outage: non-positive rejected (a
zero credits every deposit as worthless, indistinguishable from the deposit never arriving);
older than `max-rate-age` rejected; moved more than `max-rate-change-percent` **since the last
refresh** rejected — minutes apart, so a genuine market move cannot realistically trip it.

Failure policy: reuse the last accepted rate while it is still fresh; past that credit nothing
**and do not advance the watermark**, so deposits are picked up when the API returns. A delayed
credit is recoverable; a wrong one is not. The rate is fetched once per cycle so a batch is never
split across two prices.

**Open**: Phase B (gating at `CreateFile` / `buildManifest`, in-process TTL cache, fail-open) is
specified and not built.

## Operator config (2026-08-04)

Worker compensation and client billing are one-shot `coord` CLI workflows, not daemons, and have
no `enabled` setting. Existing coordinator configs may omit all `compensation.*` and `billing.*`
keys: cfgstruct supplies their built-in rates. Both need only `app-config.db.postgres-dsn`; start
the upgraded `coord core` or `coord run` once first so migrations 000033-000037 are applied.
Storage inputs depend on the normal hourly `accounting` and `worker-tally` chores having produced
snapshots. Client invoice generation also parses compensation rates to enforce that client prices
cover worker costs.

**Post-merge migration conflict fixed 2026-08-04:** merging `develop` introduced
`000033_worker_version` alongside the feature branch's old `000033_worker_compensation`, which made
the embedded golang-migrate driver reject the duplicate before telemetry started. The feature chain
was shifted to compensation=34, worker-created-at=35, payment-receipt-unique=36,
client-billing=37, wallet-sweeps=38. Verified by migrating the local database cleanly from version
30 to 38 and starting `coord core`; `chain_watermarks` exists. An empty table remains expected until
an enabled deposit watcher successfully scans its first block.
