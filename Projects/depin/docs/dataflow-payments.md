---
type: doc
project: depin
tags: [depin, dataflow, payments, billing, compensation, blockchain, aioz]
created: 2026-07-31
---

# dataflow: payments — money in, money out

How bytes become money in both directions: what a **client is charged** for storing and
downloading, and what a **worker is paid** for storing and serving. This is a **cross-cutting**
guide — it follows the two flows through every package they touch, naming the real types and
functions. For subsystem-local detail see the *Where to go next* links at the bottom.

> **Nothing here runs on a timer.** Pricing and recording are one-shot operator commands
> (`coord billing …`, `coord compensation …`). The only money-related daemon is the deposit
> watcher, and it is gated off by default. The coordinator never decides on its own to move
> funds.

---

## 1. Summary

The two flows are deliberate mirrors of each other:

```
   clients pay                                    workers earn
   ───────────                                    ────────────
   contract_storage_tallies                       worker_storage_tallies
   contract_egress_rollups                        worker_bandwidth_rollups
        │ QueryContractPeriodUsage                     │ QueryWorkerPeriodUsage
        ▼                                              ▼
   internal/billing                               internal/compensation
     storage + segments + egress x rates            at-rest + per-action x rates
     - free-egress allowance                        x surge, - withholding, + disposed
        ▼                                              ▼
   client_invoices                                 worker_paystubs
   client_ledger  (append-only, signed)            worker_payments
   client_balances (denormalised)
        ▲
   client_deposits ◄── coord/deposit watches the chain and credits
        ▲
   custodial address ◄── client sends AIOZ        ──► swept to the root wallet
```

Both sides share the same conventions, because they are the same problem pointed in opposite
directions: exact integer money (`vo.USD`, attousd), exact `decimal` byte-hours, a shared
calendar period, and "re-running a period is a correction, not a second transfer".

---

## 2. Metering (both sides)

`coord/accounting` is the only meter. A chore snapshots usage; byte-hours are integrated at
**query** time by pairing consecutive snapshots — the coordinator never re-scans history to bill.

| Written by | Table | Holds |
| --- | --- | --- |
| `tally.Service` (Core role, hourly) | `contract_storage_tallies` | Σ encrypted size, segment count, object count, per contract |
| `workertally.Service` (Core role, hourly) | `worker_storage_tallies` | Σ derived piece size, per worker |
| `coord/file` at manifest build | `contract_egress_rollups` | bytes the client asked to reconstruct |
| `coord/order` at settlement | `worker_bandwidth_rollups` | allocated / settled / dead, per action |

Two reads feed money, and they differ from the reporting reads in ways that matter:

- **half-open `[from, to)`** — the reporting queries use `BETWEEN`, which is inclusive at both
  ends; across two adjacent periods that pays and charges the boundary hour twice;
- **exact `decimal` byte-hours** — a petabyte held a month is ~7e17 byte-hours, past where a
  float64 represents every integer, and the figure is multiplied by a rate;
- **idle rows included** — a zero bill or zero payout is visibly zero, not a missing row.

An emptied contract gets an explicit **zero** tally, without which the last non-zero value
would keep accruing forever.

---

## 3. Client billing

`internal/billing` prices three dimensions and aggregates per account, because invoices are
per contract but money is per account:

```
storage  = byte-hours    / 1e12 / 720 x rate-storage-tb-month   (default $4.00)
segments = segment-hours / 720        x rate-segment-month      (default $0.0000088)
egress   = bytes         / 1e12       x rate-egress-tb          (default $25.00)
         - free-egress allowance (off by default)
```

A month is a flat **720 hours**, not the real length of the month: a customer quoted "$4 per
TB-month" expects the same price in February and March.

The defaults are set against what the network *pays*: at-rest costs $0.00000208/GB-hour =
**$1.4976/TB-month**, egress costs **$20.00/TB**. `Policy.CheckMargin` compares the two and
`coord billing generate-invoices-csv` refuses to run while a client rate is below the worker
cost — pricing below cost can be a deliberate promotion, but it should not happen by accident
and be discovered from the ledger. The check lives in `cmd/coord` because both pricing
packages are leaves and neither can see the other's config.

**Segments are a real cost driver.** A million 1 KiB objects cost the coordinator far more
metadata than one 1 GiB object holding the same bytes, so segment-hours are billed separately
from byte-hours.

### Operator flow

```
coord billing generate-invoices-csv 2026-07 --output invoices.csv   # priced, nothing charged
# read the file
coord billing record-period invoices.csv                            # one transaction, accounts debited
coord billing balances --verify-ledger
```

Splitting price from charge is the point: the first time anyone sees a bill should not be
after the account was debited.

---

## 4. Worker compensation

`internal/compensation` prices per **action** (each has its own rate), then applies the
lifecycle rules:

```
total = Σ(usage x rate)  ×surge  − held(age)  + disposed(vested)
      zeroed entirely if disqualified, or if never seen during the period
```

Withholding is a bond against abandonment: a worker that vanishes early forfeits what is still
held, so the network is not paying up front for storage that is not kept. `codes` on a paystub
explains the row — `D` disqualified, `O` offline, `E` in-withholding, `X` graceful exit.

Workers are paid on **settled** bytes, never allocated: the difference is the dead over-fetch
nobody owes for. Clients, by contrast, are charged on the manifest — see the biases below.

`workers` has **no payout address column**, deliberately. A worker supplies an address when it
withdraws and it is recorded on that payment, so the network holds no standing instruction
about where anyone's money goes.

---

## 5. Money in: deposits

`coord/deposit` is the only part of the system that talks to a chain.

Each client gets a custodial AIOZ address at registration (`coord/wallet`). The watcher scans
the **EVM JSON-RPC** for value transfers into those addresses, prices each at a live AIOZ/USD
rate, and credits the account exactly once — keyed on `(tx_hash, event_index)`, so re-scanning
a block credits nothing.

> **Only EVM transfers are credited.** On this chain the EVM view does not include
> Cosmos-native transactions — six consecutive mainnet blocks that CometBFT reported as
> carrying transactions all reported zero over the EVM RPC. A client funding from a Cosmos
> wallet is not credited automatically; the funds are in an address the network controls, and
> crediting them is a manual `coord billing credit`. **Tell clients to deposit from an EVM
> wallet.**

The rate comes from a price API and is **stamped on the deposit row**, so what a deposit was
worth is fixed at the moment it landed. Three guards sit between the API and the money: reject
non-positive (a zero credits every deposit as worthless), reject quotes older than
`max-rate-age` by the API's own timestamp, and reject a move over
`max-rate-change-percent` **between consecutive refreshes** — minutes apart, so a real market
move cannot realistically trip it but a garbled response will. If no usable rate can be
established the cycle credits nothing and does not advance its watermark: a delayed credit is
recoverable, a wrong one is not.

## 6. Money out: sweeps

`coord deposit sweep` drains the custodial addresses into one root wallet. It is a **dry run
unless `--broadcast`** — the reading an operator gets by accident has to be the harmless one.

The custodial keys are `ethsecp256k1`, the same curve Ethereum uses, so a sweep is an ordinary
signed EVM transfer. A freshly funded address holds AIOZ and nothing else, so it pays its own
fee out of the balance: it sends `balance − gasLimit × gasPrice`, and an address below that
threshold is reported as *skipped* rather than failed.

**A sweep never touches `client_balances`.** The client was credited when the deposit landed;
a sweep moves the network's own custody between addresses it controls. Recording it against a
client balance would double-count. `wallet_sweeps` is an audit trail and nothing more.

---

## 7. Things to know before trusting a number

- **Client egress is metered at manifest build, not delivery.** An aborted download is billed
  in full; a CDN or `304` hit is not billed at all. Reconciling against settled orders is not
  possible today because an `OrderLimit` carries a worker id but no contract or owner id.
- **Ingress is not billed.** Uploads record worker-side allocation but nothing contract-side —
  a client pays to keep data and to take it back out, not to put it in.
- **Nothing is gated.** A balance at or below zero does not stop uploads or downloads yet. The
  balance and the accrual query exist so the check can be added at `CreateFile` and
  `buildManifest`; that phase deliberately came after metering, so real numbers could be
  checked against reality before money could break a customer's upload.
- **Both ledgers start empty** and are authoritative from their first recorded period. Nothing
  before them is reconstructed.
- **Idempotency is structural, not conventional**, because re-running an operator command is
  normal: invoices upsert on `(period, contract)`; a charge is unique per `(owner, period)`; a
  deposit is unique per `(tx_hash, event_index)`; a worker payment is unique per
  `(worker, period, receipt)`. That last one was added *after* a repeated `record-period`
  doubled what the network believed it had paid — every automated test passed, and only a
  manual re-run caught it.

---

## 8. Where to go next

- [`coord/accounting/README.md`](../coord/accounting/README.md) — the meter, and why the two
  period queries are half-open and exact
- [`internal/billing/README.md`](../internal/billing/README.md) — client pricing, the ledger,
  and the caveats above in full
- [`internal/compensation/README.md`](../internal/compensation/README.md) — worker pricing,
  withholding and the payout ledger
- [`coord/deposit/README.md`](../coord/deposit/README.md) — the chain reader, the rate guards
  and the sweeper; **read before operating**
- [`coord/wallet/README.md`](../coord/wallet/README.md) — custodial addresses, and the hex vs
  bech32 trap
- [`coord/billing/README.md`](../coord/billing/README.md) — the client-facing read RPCs
- [`coord/db/README.md`](../coord/db/README.md) — the three money repositories and why they
  are kept apart
