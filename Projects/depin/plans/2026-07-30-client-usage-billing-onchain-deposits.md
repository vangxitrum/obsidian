# Client usage billing, funded by on-chain AIOZ deposits

## Context

Workers now get paid for what they do (`internal/compensation`, `coord compensation`, branch
`feat/worker-reward`). Nothing charges the other side. Reading `../storj/client-usage.md` against
depin shows the same situation Phase 1 of the worker work found: **the meter already exists, the
money does not.**

Already collected, hourly, by `coord/accounting/tally` (core role, `Interval` default 1h):

- `contract_storage_tallies(contract_id, interval_start, total_bytes, segment_count, object_count)`
  - point-in-time snapshots, byte-hours integrated at query time by
    `accounting.IntegrateByteHours` (`coord/accounting/accounting.go:252`, exact `decimal`)
  - zero-prepopulated for emptied contracts (`tally/tally.go:50`) so an emptied contract bills to zero
- `contract_egress_rollups(contract_id, interval_start, egress_bytes)` - written at manifest-build
  time by `recordDownloadBandwidth` (`coord/file/endpoint.go:921`)

Missing, entirely: any client price list, any invoice, any balance, any way for a client to pay.
`wallets.balance` exists and is inert; `wallet.InsufficientBalanceErr` is declared and never
referenced; no chain client exists anywhere in the repo.

Storj's answer is Stripe. Ours is on-chain only. Decisions taken for this plan:

| Decision | Choice |
| --- | --- |
| Settlement | **Prepaid balance**: deposit credits it, a billing period debits it |
| Unit of account | **USD** (`vo.USD`, attousd) - an AIOZ deposit is converted at a rate recorded on the deposit row |
| Deposit path | **Per-client custodial address** - `coord/wallet` already mints one per client; a chore watches the chain for transfers to it |
| Phase A scope | Meter, price, invoice, deposit, balance, CLI. **No gating yet** - Phase B, specified at the end |

Intended outcome: at the end of a month an operator can run one command, get a per-contract
invoice, debit it from a balance that real on-chain AIOZ funded, and see what every account owes -
with the same idempotency discipline the worker ledger ended up needing.

## What this reuses (do not rebuild)

| Need | Existing thing |
| --- | --- |
| Exact money type | `internal/vo/usd.go` - `vo.USD`, attousd, `NUMERIC(60,0)`, single rounding boundary in `NewUSDFromDecimal` |
| Exact token type / address | `internal/vo/aioz_coin.go` (attoaioz), `internal/vo/address.go` (`AccAddress`, bech32 **and** hex) |
| Chain constants | `pkg/process/blockchain.go` - `Bech32Prefix "aioz"`, `DefaultDenom "attoaioz"`, denom registration |
| Package shape to mirror | `internal/compensation/` - `rates.go` / `config.go` / `statement.go` / `db.go` / `invoice.go`, leaf discipline, CSV contract |
| Ledger shape to mirror | `coord/db/compensation_repo.go`, migrations `000033`/`000035` |
| CLI shape to mirror | `cmd/coord/compensation.go` + registration at `cmd/coord/main.go:162` |
| Usage query shape to mirror | `AccountingRepository.QueryWorkerPeriodUsage` (`coord/db/accounting_repo.go:564`) - three queries joined in Go, half-open range |
| Deposit key material | `coord/wallet/service.go:41` `CreateWallet` (ethsecp256k1, AES-encrypted privkey), `wallets` table |
| Chore wiring | `coord/peer.go:541` `setupAccountingTally` on the **core** role, `sync2.NewCycle` |

Key identity mapping, confirmed in migrations `000002`/`000003`:
`clients.id` UUID **PK** = the mTLS peer identity = the account key;
`clients.address` unique = the AIOZ account address; `contracts.owner_id -> clients.id`;
`wallets.owner -> clients.address`. The ledger keys on `clients.id`; the watcher resolves
`wallets.address -> wallets.owner -> clients.id`.

---

## WS0 - Fix the client meter before pricing it

### 0a. A half-open period query

Add to `accounting.Store` (`coord/accounting/accounting.go:129`) and
`coord/db/accounting_repo.go`:

```go
QueryContractPeriodUsage(ctx, from, to time.Time) ([]ContractPeriodUsage, error)
```

returning per contract: `ContractID`, `OwnerID`, `StorageByteHours decimal.Decimal`,
`SegmentHours decimal.Decimal`, `EgressBytes int64`. Three queries joined in Go exactly like
`QueryWorkerPeriodUsage`: contracts (for `owner_id`, so a zero-usage contract still gets a row),
integrated storage tallies, summed egress.

**The range must be half-open `[from, to)`.** Every existing client-side query
(`ListContractTallies`, `ListContractEgress`, `ListAccountEgress`,
`accounting_repo.go:292,420,441`) uses inclusive `BETWEEN`, which bills the boundary hour to both
adjacent months. `QueryWorkerPeriodUsage` already documents this at `accounting.go:210`. Leave the
existing reporting queries alone - they are not billing.

### 0b. Segment-hours

`contract_storage_tallies.segment_count` is already collected and never read back.
`StorageTally` (`accounting.go:60`) gains `SegmentCount int64`, `ListContractTallies` selects it,
and `IntegrateByteHours` gets a sibling that integrates the segment series (or is generalized over
a field selector - either, but keep the exact-`decimal` accumulation). Segment count is a real
cost driver: many tiny objects cost the coordinator metadata regardless of bytes.

### 0c. Known metering bias - document, do not price around

Client egress is recorded when the **manifest is built**, not when bytes are delivered
(`coord/file/endpoint.go:921`, best-effort and non-fatal at `:936`). Consequences that must go in
the README rather than be silently baked into a bill:

- an aborted download is billed in full;
- an edge `HEAD` (`edgeserver/handler.go:269`) and the suffix-range probe (`:200`) each cost one
  resolve, so a 206 bills two manifests (1 byte extra - negligible, but real);
- a CDN or 304 hit (`handler.go:97`) bills nothing at all;
- uploads bill nothing - ingress is deliberately unbilled (`endpoint.go:282`), matching Storj.

Reconciling client egress against **settled** orders (the honest fix, and what the accounting
README at `coord/accounting/README.md:23-26` already wrongly claims happens) is impossible today:
`OrderLimit` carries no contract or owner id (`coord/order/signer.go:211-227`), so settlement can
only ever attribute to a worker. That is a separate change. Correct the README's claim as part of
this work.

---

## WS1 - `internal/billing`, a leaf package

Mirrors `internal/compensation` file for file. Depends on stdlib, `shopspring/decimal`,
`internal/vo`, `internal/period` only. The accounting -> billing mapping lives at the CLI call
site, never inside the package.

- **`internal/period`** (new, tiny): move `internal/compensation/period.go` here and leave
  `type Period = period.Period` aliases behind in `compensation` so no existing call site changes.
  Both ledgers need the same YYYY-MM type and its `MarshalCSV`/`UnmarshalCSV`.

- **`rates.go`** - `Rates{StorageTBMonth, EgressTB, SegmentMonth decimal.Decimal}`, plus
  `StorageCost(byteHours)`, `EgressCost(bytes)`, `SegmentCost(segmentHours)`. Same decimal-TB /
  decimal-GB conventions as `compensation/rates.go:11`.

  Proposed defaults, and why:

  | Knob | Default | Reasoning |
  | --- | --- | --- |
  | `rate-storage-tb-month` | `4.00` | worker at-rest cost is `$0.00000208`/GB-h = **$1.4976/TB-month**, so ~2.7x |
  | `rate-egress-tb` | `25.00` | worker `rate-get-tb` is **$20.00**/TB; anything below that loses money on every byte served |
  | `rate-segment-month` | `0.0000088` | Storj's segment fee; covers metadata cost of many tiny objects |

  **Invariant worth enforcing**: client egress rate must be >= the worker `rate-get-tb`. The check
  cannot live in either leaf package (neither sees the other's config); put it in
  `cmd/coord/billing.go`, where both `coord.Config.Billing` and `coord.Config.Compensation` are
  bound, and fail the command rather than generate a loss-making invoice.

- **`config.go`** - `Config` of `help:`/`default:`-tagged **strings** + `Parse() (Policy, error)`,
  exactly like `compensation/config.go:16`. Includes `EgressDiscountRatio` (Storj's
  `applyEgressDiscount`: free egress up to N x stored bytes), **default `0` = disabled**, so the
  knob exists without silently discounting.

- **`invoice.go`** - `ContractUsage` (input), `Invoice` (per contract: usage columns, per-dimension
  cost, discount, total), `GenerateInvoices(policy, period, usages) []Invoice`, and
  `AggregateByOwner(invoices) []AccountCharge`. CSV read/write with a strictly validated header and
  a byte-identical round-trip test, per `compensation/invoice.go:270`.

- **`db.go`** - `Invoice`, `LedgerEntry`, `Deposit`, `Balance`, and the `DB` interface the
  coordinator implements.

---

## WS2 - Ledger and deposit schema

One migration, `coord/db/migrations/000036_client_billing.{up,down}.sql`
(`make coord/create NAME=client_billing` - **override `COORD_DB_PATH=./coord/db/migrations`**, the
Makefile default `./coord/infrastructure/db/migrations` is stale).

| Table | Key | Purpose |
| --- | --- | --- |
| `client_invoices` | PK `(period, contract_id)`, index `(owner_id, period)` | what was computed. Usage as `NUMERIC(60,6)`, every cost + total as `NUMERIC(60,0)` attousd |
| `client_ledger` | `id BIGSERIAL`, index `(owner_id, created_at)` | append-only money movement: `kind` in `deposit`/`charge`/`adjustment`, **signed** `amount NUMERIC(60,0)`, nullable `period`, `reference`, `notes` |
| `client_balances` | PK `owner_id` | `balance NUMERIC(60,0)`, updated in the *same transaction* as every ledger insert - so Phase B gating is one indexed read, not a growing `SUM` |
| `client_deposits` | `id BIGSERIAL`, **`UNIQUE (tx_hash, event_index)`** | one row per observed on-chain transfer: `deposit_address`, `block_height`, `block_time`, `amount_aioz NUMERIC(60,0)`, `usd_rate NUMERIC(40,18)`, `amount_usd NUMERIC(60,0)`, `credited_at` |
| `chain_watermarks` | PK `name` | `last_height BIGINT` - the watcher's resume point |

Idempotency, learned the hard way on the worker ledger (a repeated `record-period` doubled
`TotalPaid`, and **only the manual re-run caught it** - every automated test passed):

- invoices upsert on `(period, contract_id)` - a re-run is a correction, not a second charge;
- charges get `CREATE UNIQUE INDEX ... ON client_ledger (owner_id, period) WHERE kind = 'charge'`
  + `ON CONFLICT DO NOTHING`, so re-running a closed period cannot debit twice;
- deposits get `ON CONFLICT (tx_hash, event_index) DO NOTHING`, so a re-scanned block credits nothing.

Also drop the inert `wallets.balance` column and mark the proto field reserved in
`pkg/pb/coord/wallet/v1/wallet.proto`. Nothing reads it; a column named `balance` that is not the
balance is a footgun once a real one exists. The down migration restores it.

`coord/db/billing_repo.go` implements `billing.DB` following `compensation_repo.go`: gorm row
structs with `TableName()`, a `var _ billing.DB = (*BillingRepository)(nil)` assertion,
`defer mon.Task()(&ctx)(&err)` per method, and `RecordPeriod(invoices, charges)` as **one
transaction**. Add a `VerifyBalances` query (`client_balances` vs `SUM(client_ledger.amount)`)
because a denormalized balance that nobody checks is a balance nobody should trust.

---

## WS3 - Deposit watcher (`coord/deposit`)

A new chore on the **core** role only, next to the tally cycles (`coord/peer.go:541`), gated
`Enabled: false` by default exactly like `RepairConfig`/`GCConfig` (`coord/config.go:59-72`).

Config: `NodeRPC` (CometBFT RPC URL), `Interval` (default `30s`), `StartHeight`,
`MinConfirmations` (default `1` - Cosmos has instant finality), `BatchBlocks`, and
`AiozUsdRate` (a fixed decimal string for Phase A).

Cycle:

1. read `chain_watermarks`, load the deposit-address set from `wallets` (refreshed each cycle, so a
   client registered a minute ago is watched);
2. for each height up to `latest - MinConfirmations`, fetch block results and collect `transfer`
   events whose `recipient` is a known deposit address and whose denom is `attoaioz`;
3. per event, in one transaction: insert `client_deposits` (conflict -> skip), convert at the
   configured rate into `amount_usd`, append a `deposit` ledger entry, bump `client_balances`;
4. advance the watermark.

**The rate is recorded on the deposit row, never re-derived.** A later oracle changes what new
deposits are credited at; it must not restate history.

Declare the `Store` interface in `coord/deposit`, implement it in `coord/db` - the established
direction (subsystems never import `coord/db`).

**Verify against a live AIOZ node before trusting step 2**: an AIOZ address has both a bech32 and a
hex form (`vo.AccAddress`), and this network runs Ethermint. Confirm whether a transfer sent to the
`0x` form via the EVM surfaces as a bank `transfer` event in `block_results`. If it does not, an
`eth_getBlockByNumber`/`eth_getLogs` scan path is required and a whole class of deposits would
otherwise be silently lost. Do not assume either way.

Out of scope, stated plainly: sweeping deposited AIOZ from custodial addresses to a treasury (needs
the AES-encrypted key and a signing path), withdrawing unspent balance, and any live price oracle.

---

## WS4 - CLI and client-facing RPCs

`cmd/coord/billing.go`, registered like `compensation.go` (`cmd/coord/main.go:162`). **Bind each
leaf command separately** - the gotcha documented at `cmd/coord/main.go:228-243`: viper only
repopulates the config of the command that actually runs, so binding the parent leaves rates at
zero.

| Command | Behaviour |
| --- | --- |
| `coord billing generate-invoices-csv <YYYY-MM> [--output]` | `QueryContractPeriodUsage(period.StartDate(), period.EndDateExclusive())` -> `GenerateInvoices` -> CSV. Fails if egress rate < worker `rate-get-tb`. |
| `coord billing record-period <invoices.csv>` | one transaction: upsert invoices, insert per-owner charges, update balances |
| `coord billing credit <owner-id> <amount-usd> [--reference] [--notes]` | manual adjustment / refund / promo |
| `coord billing balances [--owner]` | balance, total deposited, total charged |

New `pkg/pb/coord/billing/v1/billing.proto` + `coord/billing/endpoint.go` on the **api** role,
authenticated with `identity.PeerIdentityFromContext` exactly like `coord/accounting/endpoint.go:40`:

- **`GetDepositAddress`** -> the caller's custodial AIOZ address in bech32 and hex. This is the
  gap that makes everything else usable: a wallet is minted at `CreateClient`
  (`coord/client/service.go:66`) and its address has never been returned to anyone. Needs a
  `GetByOwner` on `wallet.Store`, which today has only `Create` (`coord/wallet/store.go:7`).
- `GetBalance` -> balance attousd + last invoiced period.
- `ListInvoices(from, to)`, `ListDeposits(from, to)`.

Config wiring: `Billing billing.Config` and `Deposit deposit.Config` in `coord/config.go:27`.

`internal/billing/README.md` mirrors the compensation README, including its "things to know before
trusting a number" section: the WS0c egress bias, no gating yet, the fixed FX rate, and that the
ledger starts empty with **no backfill** of pre-ledger usage.

---

## Files

**New**
- `internal/period/period.go` (moved), alias shim in `internal/compensation/period.go`
- `internal/billing/{rates,config,invoice,db}.go` + `README.md`
- `coord/deposit/{service,store,config}.go`
- `coord/billing/endpoint.go`, `pkg/pb/coord/billing/v1/billing.proto`
- `coord/db/{billing_repo,deposit_repo}.go`
- `coord/db/migrations/000036_client_billing.{up,down}.sql`
- `cmd/coord/billing.go`

**Modified**
- `coord/accounting/accounting.go` - `ContractPeriodUsage`, `StorageTally.SegmentCount`, segment-hour integration, `Store` interface
- `coord/db/accounting_repo.go` - `QueryContractPeriodUsage`, `ListContractTallies` selects `segment_count`
- `coord/accounting/README.md` - correct the "charged on settled egress" claim
- `coord/config.go`, `coord/peer.go` (deposit chore on core), `coord/core.go`, `coord/api.go` (billing endpoint), `cmd/coord/main.go`
- `coord/wallet/store.go` + `coord/db/wallet_repo.go` - `GetByOwner`
- `pkg/pb/coord/wallet/v1/wallet.proto` - reserve the dropped `balance` field

---

## Verification

1. **Unit** - `go test ./internal/billing/... ./internal/period/...`
   - 1 TB stored for a 720-hour month at `4.00` prices to exactly `$4.00`; 1 TB egress at `25.00`
     to `$25.00`; segment-hours to segment-months.
   - CSV round-trip byte-identical, header strictly validated.
   - Egress discount off by default; on, it subtracts a storage-proportional allowance and never
     goes negative.
2. **DB integration** - `coorddbtest.Run`, live `coord-db` Postgres, `DEPIN_TEST_POSTGRES`
   - **Write this first and watch it fail against a `BETWEEN` implementation**: two adjacent
     periods over the same tallies must sum to the single-window total, not more.
   - `RecordPeriod` run twice moves the balance once (invoices upsert, charge conflicts away).
   - A re-observed deposit `(tx_hash, event_index)` credits nothing.
   - `client_balances` equals `SUM(client_ledger.amount)` after every mutation.
3. **Watcher unit** - fake block-results source: only known addresses and only `attoaioz` are
   credited; the watermark resumes mid-range; a replayed block is a no-op.
4. **End-to-end** - `internal/testplanet`, patterned on the tag-usage test
   - upload and download a file, drive two deterministic tally snapshots via `tally.Service.SetNow`
     (not wall clock, so byte-hours integrate to an exact value), generate invoices over that exact
     window and assert against a hand-computed figure;
   - insert a deposit through the repository (not the chain) and assert
     `balance == deposit - invoice`;
   - `GetDepositAddress` returns the address `CreateClient` minted.
5. **Manual, on the dev cluster** - `coord billing generate-invoices-csv`, then run
   `record-period` **twice** and confirm the balance moved once. The worker ledger's
   double-recording bug passed every automated test and was caught only by a manual repeat.

### Environment gotchas that will bite

- This worktree needs `../go-sdk` symlinked; `go.mod` points at the local path and must be
  re-pinned before committing.
- `AddContractEgress` truncates `interval_start` to the hour - test windows need >= 1h of slack
  around a known event time.
- `internal/testplanet` seeds two placements; filter `ListPlacements()` by `Name == "testplanet"`,
  do not take index 0.
- Three pre-existing test failures are unrelated to this work (edgeserver metadata headers,
  `TestPieceIDScanNullAndEmpty`, a `worker/pieces` build failure).

---

## Phase B (deferred, specified so it is not lost)

Gating, once real numbers have been checked against reality:

- **Upload** - `CreateFile` (`coord/file/endpoint.go:87`) is the one upload site with the contract
  in hand (`req.ContractId`, resolved at `coord/file/service.go:77`). `BeginSegment`/
  `CommitSegment`/`MakeInlineSegment` carry only `FileUUID`/`OwnerID` in the signed StreamID/
  SegmentID, so a per-segment check would need a `files` lookup or a `ContractID` added to
  `StreamIDContent`.
- **Download** - `buildManifest` (`coord/file/endpoint.go:793`) already resolves `file.ContractId`
  at `:798`, right where egress is recorded at `:855`. `GetDownloadInfoByTicket` never reads a peer
  identity, so the contract is the only attribution available - which is correct here: the file's
  owner pays for its delivery.
- **The check** - `balance - cost(usage since last closed period)` <= 0 => `codes.ResourceExhausted`.
  The accrual reuses `QueryContractPeriodUsage(lastClose, now)` unchanged.
- **No Redis exists in depin**, so live accounting is a per-replica in-process cache with a short
  TTL over `client_balances`, not a shared counter. Follow Storj in failing **open** on cache/DB
  error (`satellite/metainfo/validation.go:838-848`): a balance check that is down must degrade to
  no enforcement, not to an outage.
- 80%/100% threshold events, mirroring `satellite/projectlimitevents`.
