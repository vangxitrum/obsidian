	# Worker-initiated withdrawals

## Context

A worker can now be *told* what it is owed and what that is worth in AIOZ, but it has no way to
**ask for it**. Everything on the payout path is operator-driven and offline: `coord compensation
generate-invoices-csv` prices a period, `payouts-csv` builds a payment template, an operator sends
AIOZ by hand, and `record-period` writes the receipt back. The worker is never in the loop, and has
no way to see its own balance at all.

The gap is deliberate and already documented. `internal/compensation/db.go:98-107` says
`TotalAmounts.Payable()` is "the figure a worker-initiated withdrawal debits", and
`internal/compensation/README.md:145-151` states outright: *"`workers` has no payout-address column,
deliberately. A worker supplies an address when it withdraws ... that RPC is not built yet and needs
no schema change when it is."* `worker/peer.go:592` carries a matching `// TODO: payout, ...`.

This plan builds that RPC and the two command surfaces around it: a worker asks, an operator settles.

**What exists to build on.** `Payable()` already computes the withdrawable balance.
`worker_payments.to_address` (`000034:72-78`) already exists to record where money went; `amount_aioz`
/ `usd_rate` (`000044`) already record what it was worth. mTLS peer identity already authenticates
workers on four coord endpoints. `resolveAiozRate` (`cmd/coord/compensation.go:238-294`) already
resolves the payout rate. Almost none of this is new money math - it is a request lifecycle wrapped
around machinery that works.

**What does not exist.** No withdrawal table, no worker-facing payout RPC, no worker CLI that dials
the coordinator at all (`cmd/worker/main.go:77` registers only `run`, `api`, `setup`), and - relevant
to what this plan deliberately does *not* do - no treasury signing key anywhere in coord. The only
transaction the repo broadcasts is `coord/deposit/sweeper.go:277`, which signs with a *client's*
custodial key and sends to a fixed configured address.

### Decisions locked with the user

| Decision | Choice |
|---|---|
| Surface | Worker CLI (`aioznode withdraw`, `aioznode balance`) + a new worker-facing coord RPC |
| On-chain | **Record only.** Coord broadcasts nothing. A `Sender` seam is left for phase 2 |
| Data model | New `worker_withdrawals` table with explicit states, migration `000045` |
| Amount | Partial allowed via `--amount`, with a configurable minimum |
| Rate timing | **Priced at settle only.** No AIOZ quote is stored on the request |
| Settle flow | Operator-driven: the request reserves, `settle` writes the `worker_payments` row |

Two consequences of "priced at settle only" shape everything below:

- **The withdraw endpoint needs no price source.** No `deposit.RateSource`, no `TreasuryReader`, no
  `internal/fx` call, no `quoted_*` columns. The rate is resolved once, in the operator's `settle`
  command, reusing `resolveAiozRate` unchanged. That removes the endpoint's entire dependency on the
  deposit ledger and its failure modes.
- **The payout service is USD-only, deliberately and completely** - see the next section.

### The worker is not told the AIOZ rate

An explicit product decision, for now. Nothing on the worker-facing surface carries an AIOZ amount, a
USD/AIOZ rate, or anything a rate can be derived from:

| Surface | Rule |
|---|---|
| `GetPayoutBalanceResponse` | USD fields only. No `available_aioz`, no `aioz_usd_rate`, no `rate_source`, no `denom`. |
| `worker_withdrawals` | No `quoted_aioz` / `quoted_usd_rate` / `quote_source` columns exist to leak. |
| `Withdrawal` (proto, worker-visible) | id, requested_at, amount (USD), to_address, status, resolved_at, reason. **No `receipt`, no `amount_aioz`, no `usd_rate`, no `payment_id`.** |
| `aioznode balance` / `aioznode withdraw` | Print dollars only. No AIOZ line, no rate line, not even "unavailable". |

Two traps this closes that the obvious design walks into:

- **The receipt is a rate oracle.** A transaction hash lets a worker read the AIOZ amount off chain
  and divide by the USD it was told - the payout rate, exactly. Returning the receipt "so the worker
  can verify" hands over the number this decision withholds.
- **`BillingService.GetAiozPrice` already answers a worker's mTLS session.** It does no identity
  lookup by design (`coord/billing/endpoint.go:224-225`: "a rate is a fact about the network, not
  about an account"). It serves the *spot* rate, not the treasury blend workers are paid at, so it
  does not leak the payout rate - but it is the obvious thing for someone to point a worker at. Do
  not document it as the workaround, and do not add a worker-facing rate RPC "for convenience".

**State the limitation honestly rather than overselling it.** This is opacity in the API, not secrecy
in fact: a worker watches AIOZ arrive at an address it chose and knows the USD figure it was owed, so
it can compute the rate itself. What the decision actually buys is that the coordinator does not
*publish* a rate a worker could quote back, dispute against, or time a withdrawal around. The cost is
that a worker cannot tie a payment to an on-chain transfer through the API, which is a real support
burden and the first thing to revisit when this is relaxed. Both sentences belong in
`coord/payout/README.md`.

Operator surfaces are unaffected: `coord compensation withdrawals list` and the `worker_payments` row
carry the full AIOZ amount, rate, and receipt.

### The one invariant everything serves

> **A dollar is counted in exactly one of three places at every instant: unclaimed (`Payable()`),
> reserved (a pending withdrawal), or paid (a `worker_payments` row). Every transition between those
> places happens inside one database transaction.**

### The fact the whole design bends around

`Payable()` = `TotalOwed - TotalPaid`, and it moves **only** when a `worker_payments` row is written.
A withdrawal *request* writes no payment, so `Payable()` does not change when a request is accepted.
Two requests a millisecond apart both see the full balance and both pass any read-then-write check the
application can make. Everything below that looks like belt-and-braces exists because of that
sentence.

---

## 1. Migration `000045_worker_withdrawals`

`000044` is head. New table plus one column on `worker_payments`.

```sql
CREATE TABLE IF NOT EXISTS worker_withdrawals
(
    id              BIGSERIAL      PRIMARY KEY,
    worker_id       UUID           NOT NULL,
    requested_at    TIMESTAMPTZ    NOT NULL,
    -- idempotency token, one per `aioznode withdraw` invocation
    request_key     UUID           NOT NULL,
    -- canonicalised through vo.AccAddress before storing, so bech32 and hex input
    -- for the same account are byte-identical here
    to_address      TEXT           NOT NULL CHECK (to_address <> ''),
    -- attousd. The obligation, and the only money figure on this row.
    amount          NUMERIC(60, 0) NOT NULL CHECK (amount > 0),
    status          TEXT           NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'settled', 'rejected', 'cancelled')),
    resolved_at     TIMESTAMPTZ,
    payment_id      BIGINT         REFERENCES worker_payments (id),
    -- why it was rejected or cancelled; empty otherwise
    reason          TEXT           NOT NULL DEFAULT '',

    CONSTRAINT worker_withdrawals_resolution_consistent CHECK (
        (status = 'pending' AND payment_id IS NULL     AND resolved_at IS NULL)     OR
        (status = 'settled' AND payment_id IS NOT NULL AND resolved_at IS NOT NULL) OR
        (status IN ('rejected','cancelled')
                            AND payment_id IS NULL     AND resolved_at IS NOT NULL)
    )
);

-- THE RESERVATION LOCK.
CREATE UNIQUE INDEX idx_worker_withdrawals_one_pending
    ON worker_withdrawals (worker_id) WHERE status = 'pending';

CREATE UNIQUE INDEX idx_worker_withdrawals_request_key
    ON worker_withdrawals (worker_id, request_key);
CREATE INDEX idx_worker_withdrawals_worker_requested
    ON worker_withdrawals (worker_id, requested_at DESC);
CREATE INDEX idx_worker_withdrawals_queue
    ON worker_withdrawals (requested_at) WHERE status = 'pending';

ALTER TABLE worker_payments
    ADD COLUMN IF NOT EXISTS withdrawal_id BIGINT REFERENCES worker_withdrawals (id);
CREATE UNIQUE INDEX idx_worker_payments_withdrawal
    ON worker_payments (withdrawal_id) WHERE withdrawal_id IS NOT NULL;
```

The header comment must carry five things, in the house style of `000034` / `000044`:

- Why intent and transfer are separate tables - the same split `worker_paystubs` / `worker_payments`
  already makes. "We owe this", "somebody asked for it" and "it was sent" are three states, not one
  optimistic number.
- Why `idx_worker_withdrawals_one_pending` is load-bearing rather than decorative (the `Payable()`
  fact above), and that **if the one-pending rule is ever relaxed, an explicit
  `SELECT id FROM workers WHERE id=? FOR UPDATE` must be in place first** - the pattern
  `reputation_repo.go:68` and `audit_repo.go:354` already use - not after.
- Why `status` is TEXT + CHECK rather than an enum type: phase 2's `'settling'` state must be a
  constraint change and nothing else.
- **Phase 2 warning.** When `'settling'` lands, `idx_worker_withdrawals_one_pending` and every
  reserved-balance sum must widen to `status IN ('pending','settling')`. A settling row that stopped
  counting as reserved would let a worker withdraw the same money again while the first transfer sat
  in the mempool. **This is the highest-severity latent bug in the feature.**
- Why `withdrawal_id` on `worker_payments` is needed even though `000036` exists - see §4.

Down migration: drop the payments index and column first, then the table.

**Deployment order matters.** `coord api` does not run migrations (`cmd/coord/main.go:394-397,444`).
`000045` must be applied by a singleton `run`/`core` before any api replica serves the new RPCs, or
every call fails on a missing table. Note it in `docs/RELEASE.md`.

## 2. `internal/compensation` - new leaf types

New file `internal/compensation/withdrawal.go`. Leaf discipline holds: `context`, `time`,
`internal/vo`, `shopspring/decimal` only - the same import set `db.go` uses.

```go
type WithdrawalStatus string
const (
    WithdrawalPending   WithdrawalStatus = "pending"
    WithdrawalSettled   WithdrawalStatus = "settled"
    WithdrawalRejected  WithdrawalStatus = "rejected"
    WithdrawalCancelled WithdrawalStatus = "cancelled"
)

type Withdrawal struct {
    ID          int64
    WorkerID    vo.UUID
    RequestedAt time.Time
    RequestKey  vo.UUID
    ToAddress   vo.AccAddress
    Amount      vo.USD
    Status      WithdrawalStatus
    ResolvedAt  *time.Time
    PaymentID   *int64
    Reason      string
}

// WithdrawableAmounts is a worker's position as a withdrawal sees it.
//
// Deliberately not folded into TotalAmounts.Payable(). That figure is a statement
// fact -- owed minus paid -- printed into the paystub CSV and read by the monthly
// payout run. Netting an unsettled intent off it would quietly change what every
// existing reader of that column means, and one of those readers moves money.
type WithdrawableAmounts struct{ Payable, Pending vo.USD }

func (w WithdrawableAmounts) Available() vo.USD    // Payable - Pending, clamped
func (w WithdrawableAmounts) Overcommitted() bool  // Pending > Payable
```

Plus `WithdrawalRequest{WorkerID, RequestKey, ToAddress, Amount (zero = everything available),
MinAmount, Now}`, `WithdrawalSettlement{ID, AmountAioz, USDRate, Receipt, Notes, Now}`, and
`WithdrawalResolution{ID, Status, Reason, Now}` for reject/cancel.

Sentinel errors use plain `errors` (this package does not import `zeebo/errs` and should not start):
`ErrWithdrawalExists`, `ErrWithdrawalNotPending`, `ErrBelowMinimum`, `ErrInsufficientBalance`, plus
`*InsufficientBalanceError{Requested, Available, Payable, Pending}` and
`*BelowMinimumError{Requested, Minimum}` which carry the figures and implement `Is`.

**`Payable()` at `internal/compensation/db.go:105` does not change.** Add a one-line cross-reference
in its doc comment pointing at `WithdrawableAmounts` and saying it deliberately does not net
reservations.

### New `WithdrawalDB` interface

Separate from `DB` because the callers differ: the monthly CLI needs `RecordPeriod` and never touches
a withdrawal; the worker-facing endpoint needs withdrawals and must never be handed `RecordPeriod`.
`coord/db.CompensationRepository` implements both, because settlement writes both tables in one
transaction and cannot be split across two repositories.

```go
QueryWithdrawable(ctx, workerID vo.UUID) (WithdrawableAmounts, error)
RequestWithdrawal(ctx, req WithdrawalRequest) (w Withdrawal, created bool, err error)
GetWithdrawal(ctx, id int64) (Withdrawal, error)
ListWithdrawals(ctx, f WithdrawalFilter) ([]Withdrawal, error)
SettleWithdrawal(ctx, s WithdrawalSettlement) (Withdrawal, Payment, error)
ResolveWithdrawal(ctx, r WithdrawalResolution) (Withdrawal, error) // reject | cancel
```

### Config

One new field on `internal/compensation/config.go`:

```go
MinWithdrawalUSD string `help:"smallest worker-initiated withdrawal the coordinator will accept, in USD" default:"1.00"`
```

cfgstruct names it `--compensation.min-withdrawal-usd`. Parsed by a **narrow**
`ParseWithdrawal() (WithdrawalPolicy, error)` returning `WithdrawalPolicy{Minimum vo.USD}`, not by
`Config.Parse()`. Same split as `ParseRate` / `ParseSweep`, and for the same reason found earlier on
this branch: the api role must not fail to boot because a payout rate it never reads is malformed,
and it must not silently accept dust because a minimum it *does* read is. Rejects unparseable,
negative, and zero. Add `DefaultMinWithdrawalUSD` beside the other defaults in `rates.go`.

## 3. `coord/db` - the reservation, done right

New file `coord/db/withdrawal_repo.go`, methods on the **existing** `*CompensationRepository`
(settlement spans both tables). `workerPaymentRow` gains `WithdrawalID *int64`.

### 3a. Availability must be ONE statement

`QueryTotalAmounts` (`coord/db/compensation_repo.go:294-321`) runs **two** queries today. Under READ
COMMITTED each gets its own snapshot, so adding a third for reservations lets a concurrent settle be
half-visible: its payment invisible while its reservation is already released. That is an
over-withdrawal with no concurrent *request* involved at all. One CTE, one snapshot:

```sql
WITH stubs AS (
    SELECT COALESCE(SUM(held),0) AS held, COALESCE(SUM(disposed),0) AS disposed,
           COALESCE(SUM(owed),0) AS owed, COALESCE(SUM(distributed),0) AS distributed
      FROM worker_paystubs WHERE worker_id = ?
), paid AS (
    SELECT COALESCE(SUM(amount),0) AS paid FROM worker_payments WHERE worker_id = ?
), reserved AS (
    SELECT COALESCE(SUM(amount),0) AS pending FROM worker_withdrawals
     WHERE worker_id = ? AND status = 'pending'
)
SELECT stubs.*, paid.paid, reserved.pending FROM stubs, paid, reserved;
```

Feed `held/disposed/owed/distributed/paid` through the existing `assembleTotals`
(`coord/db/compensation_repo.go:379`) before using it, so the `disposed > held` invariant that guards
every other reader also guards the one path that hands out money. Define the `status = 'pending'`
predicate as a named constant used by all call sites, so phase 2's widening is a single edit.

### 3b. The race is closed by the index, not by code

`idx_worker_withdrawals_one_pending` is load-bearing. The second concurrent insert blocks on the index
tuple, then raises `23505`. Add `isUniqueViolation(err, constraint)` to `coord/db/errors.go` using
`errors.As` on `*pgconn.PgError` (`Code == "23505"`, `ConstraintName == ...`) and translate it into
`ErrWithdrawalExists`, so the loser of a race gets the same answer as the loser of a read. Do **not**
set `gorm.Config{TranslateError: true}` - that changes error behaviour for every repository in the
coordinator.

Rejected alternatives and the exact failure of each:

| Alternative | Why it fails |
|---|---|
| Subtract pending in-tx, no index | Textbook write skew. Both transactions read the same snapshot, neither sees the other's uncommitted row, both insert. This is what the obvious implementation looks like. |
| `SELECT ... FROM worker_withdrawals ... FOR UPDATE` | Locks **zero rows** when the worker has no pending request. The first concurrent pair races. Most dangerous option because it looks like a fix. |
| `SELECT id FROM workers WHERE id=? FOR UPDATE` | Actually correct, and the house pattern. Rejected as *primary* only because it puts the payout path in lock contention with the audit/reputation hot path. Required before the one-pending rule is ever relaxed. |
| REPEATABLE READ | Does not detect write skew. Both commit. Worse than nothing because it looks like protection. |
| SERIALIZABLE | Correct, but needs `40001` retry plumbing that exists nowhere here, and its predicate locks over `SUM(owed) FROM worker_paystubs` would make `RecordPeriod` and withdrawals abort each other during the payout run. |

### 3c. Settle is one transaction, in this order

1. `SELECT ... WHERE id=$1 FOR UPDATE`.
2. Assert `status='pending'`, else `ErrWithdrawalNotPending`.
3. **Re-run the availability CTE, excluding this withdrawal's own reservation.** A request made in
   June and settled in August may have been overtaken by the July `record-period` run, which pays
   `Payable()` without knowing withdrawals exist. This is the only unconditional backstop against
   double-paying.
4. `INSERT INTO worker_payments (..., period=NULL, amount, to_address, receipt, amount_aioz,
   usd_rate, withdrawal_id) RETURNING id`.
5. `UPDATE worker_withdrawals SET status='settled', resolved_at, payment_id ... WHERE id=$1 AND
   status='pending'`; assert `RowsAffected == 1`.

**Step 4 must NOT go through `insertPayments`** (`coord/db/compensation_repo.go:264`). Its untargeted
`clause.OnConflict{DoNothing: true}` swallows every unique violation, so a duplicate `withdrawal_id`
would insert nothing while the settle reported success - reservation released, transfer unrecorded,
`Payable()` permanently overstated. It also leaves gorm's autoincrement id at zero on a skip, so
`payment_id` would land NULL. Use a plain `tx.Create` and let `23505` surface. Put that reason in a
comment at the call site.

Reject and cancel are `UPDATE ... WHERE id=$1 AND status='pending'` with the same `RowsAffected == 1`
assertion. No payment row; the reservation simply drops and the worker may request again.

## 4. Idempotency

| Replay | Mechanism |
|---|---|
| Worker retries after a wire timeout | `request_key`, unique on `(worker_id, request_key)`. On `23505` re-read and return the existing row with `created=false`; the CLI prints "already requested (#41)". |
| Worker invokes `withdraw` twice | `idx_worker_withdrawals_one_pending` -> `AlreadyExists`, carrying the pending request's id and amount. |
| Operator settles twice | `WHERE id=$1 AND status='pending'` + `RowsAffected == 1`. The whole transaction, payment included, rolls back. |
| Two payment rows for one withdrawal | `idx_worker_payments_withdrawal`. **This is why the new column exists:** `idx_worker_payments_receipt_unique` (`000036:17`) is `WHERE receipt <> ''`, and a phase-1 settle is done by an operator who may have no tx hash yet - exactly the set `000036` exempts. Two rows would double `TotalPaid`: the bug `000036` was written to prevent, reintroduced through the hole it left. |

## 5. Two payout paths that cannot see each other

`record-period` pays `Payable()`; the withdrawal path reserves against the same balance. A worker with
a pending $25 request can be paid its full $42 by the monthly run, then settle for $25 more. Three
guards, weakest to strongest:

1. `generate-invoices-csv` prints a **stderr warning** listing every worker with a pending request and
   the total reserved, beside the rate basis it already prints.
2. `record-period` **refuses** if any payment in the CSV is for a worker with a pending request,
   unless `--allow-pending-withdrawals`.
3. **Settle re-validates availability** (§3c step 3). Unconditional; the actual backstop. 1 and 2
   exist so the operator finds out before the money moves.

Do **not** repurpose `worker_paystubs.distributed` for this. It is per-`(period, worker)` and a
withdrawal spans periods; apportioning it is arbitrary. Update
`internal/compensation/README.md:140-142` to say `SUM(worker_withdrawals.amount) WHERE
status='pending'` is the third state, at the granularity where it means something.

## 6. Proto: `pkg/pb/coord/payout/v1/payout.proto`

**New service, not an extension of `BillingService`.** That file's safety rests on one invariant -
"the caller *is* the account" (`coord/billing/endpoint.go:30-32`) - and mixing in RPCs scoped to a
different class of identity would end it.

Package `hub.payout.v1`, `go_package = "aioz-depin/pkg/pb"`, gogoproto, all RPCs unary (satisfies the
`UNARY_RPC` lint rule in `buf.yaml`).

```proto
service PayoutService {
  rpc GetPayoutBalance(GetPayoutBalanceRequest) returns (GetPayoutBalanceResponse);
  rpc RequestWithdrawal(RequestWithdrawalRequest) returns (RequestWithdrawalResponse);
  rpc ListWithdrawals(ListWithdrawalsRequest) returns (ListWithdrawalsResponse);
  rpc CancelWithdrawal(CancelWithdrawalRequest) returns (CancelWithdrawalResponse);
}
```

`GetPayoutBalanceResponse` carries `payable`, `available`, `pending`, `total_owed`, `total_paid`,
`total_held`, `total_disposed`, `min_withdrawal` - all attousd decimal strings, matching
`GetBalanceResponse` on the client side.

`RequestWithdrawalRequest{to_address, amount_usd (empty = everything available), request_key}`.
`WithdrawalStatus` enum has PENDING / SETTLED / REJECTED / CANCELLED.

The worker-visible `Withdrawal` message is a **deliberately narrower projection** of the Go type -
`id`, `requested_at`, `amount` (attousd), `to_address`, `status`, `resolved_at`, `reason`. It omits
`payment_id`, `receipt`, `amount_aioz` and `usd_rate`. The proto comment must say why in one line, or
the next person will "fix" the omission: **a receipt is a rate oracle** - it resolves to an on-chain
AIOZ amount which, divided by the USD on the same row, is the payout rate.

Doc comments must say, in the house style of `billing.proto:11-16`: every RPC is scoped to the mTLS
peer identity so none of them take a worker id; the coordinator records a request and does not move
money; the service reports dollars only and never a rate; and at most one request may be pending per
worker, refused rather than queued, because two pending requests against one balance is how the same
dollar gets paid twice.

Then `make proto`.

### Identity scoping: carry nothing

No `worker_id` field anywhere in the service. `coord/accounting/endpoint.go:93-103` is the proof that
accepting an id is a liability - the field is only safe because someone remembered the
`if !peerIden.ID.Equal(req.WorkerId)` guard three lines later, and that guard is one refactor or one
copy-paste from being absent. `GetWorkerUsage` leaking a byte-hour count is embarrassing;
`RequestWithdrawal` missing the same guard is "anyone can drain anyone". `coord/order/settlement.go:82`
already does it the right way. A `worker_id` field is also an attractive nuisance: the first person
who wants "let support file a withdrawal on a worker's behalf" reaches for it, and then the endpoint
is pay-anyone gated on a flag.

One improvement over the billing precedent: `coord/billing/endpoint.go:77-80` returns the raw
`PeerIdentityFromContext` error, which reaches the client as `codes.Unknown`. Wrap it to
`Unauthenticated` here.

## 7. `coord/payout` - the endpoint

New package, mirroring `coord/billing` (the client-facing money service).

```go
type Endpoint struct {
    log         *zap.Logger
    ledger      compensation.DB           // QueryTotalAmounts
    withdrawals compensation.WithdrawalDB
    workers     contact.Store             // GetWorker: existence + disqualification
    policy      compensation.WithdrawalPolicy
    payoutpb.UnimplementedPayoutServiceServer
}
```

No rate source, no treasury reader - the consequence of pricing at settle.

### Validation order and gRPC codes

Cheapest first; one DB round trip only after the arguments are known-good; the balance check last
because it must be inside the transaction.

| # | Check | Code | Reason slug |
|---|---|---|---|
| 1 | mTLS peer identity present | `Unauthenticated` | - |
| 2 | address non-empty | `InvalidArgument` | `validate-failed` |
| 3 | `vo.ParseAccAddress` succeeds | `InvalidArgument` | `invalid-address` |
| 4 | **`len(addr.Bytes()) == 20`** | `InvalidArgument` | `invalid-address` |
| 5 | **not the zero address** | `InvalidArgument` | `zero-address` |
| 6 | mixed-case hex passes EIP-55 | `InvalidArgument` | `invalid-address-checksum` |
| 7 | amount parses / not negative | `InvalidArgument` | `validate-failed` |
| 8 | worker row exists | `NotFound` | `not-found` |
| 9 | worker not disqualified | `PermissionDenied` | `worker-disqualified` |
| 10 | amount >= `min-withdrawal-usd` | `FailedPrecondition` | `below-minimum` |
| 11 | amount <= available *(in tx)* | `FailedPrecondition` | `insufficient-balance` |
| 12 | no other pending request *(23505)* | `AlreadyExists` | `withdrawal-already-pending` |
| 13 | replayed `request_key` | `OK`, `created=false` | - |

Three of these are money bugs if skipped:

- **#4.** `vo.ParseAccAddress` tries bech32 first and `sdk.VerifyAddressFormat` only rejects *empty*,
  so it accepts a 32-byte address. `AccAddress.String()` then calls `common.BytesToAddress`, which
  **truncates to the last 20 bytes** - the money would go to a different account.
- **#5.** `ParseAccAddress` accepts `0x0000...0000`. Paying it burns the money.
- **#6.** `common.IsHexAddress` does no checksum validation. Nothing can catch an all-lowercase hex
  typo, which is why the CLI must echo the canonical address back in **both** encodings and require
  confirmation unless `--yes`.

**#9 is not implied by the balance.** The ledger zeroes a disqualified worker's future periods but
does not claw back past `owed`, so `Payable()` can be positive for a DQ'd worker. Bar the self-service
path only; the error should say an operator can still release the balance via
`record-one-off-payments`.

**#10 before #11** so a worker with $0.40 available is told "the minimum is $1.00", not the less useful
"insufficient balance". The floor applies to the resolved amount, including withdraw-everything.

**`amount_usd` omitted resolves server-side, inside the reserving transaction.** If the CLI computed
it from a prior balance read, a `record-period` committing in between turns a valid request into a
rejection, or into a reservation for a different amount than the CLI thinks. The response carries the
exact reserved amount; that is what the CLI prints.

### Wiring - two call sites, both mandatory

`setupPayoutEndpoint()` in `coord/peer.go` beside `setupBillingEndpoint` (`:951-961`), registering via
`b.Server.RegisterGRPCServiceFunc` so it lands on both the TCP and p2p listeners (both wrap
`tls.NewListener(..., ServerTLSConfig())`, so `PeerIdentityFromContext` works on either).

1. `coord/peer.go` - `Payout struct{ Endpoint *payout.Endpoint }` field on `Peer` beside `Billing`
   (`:1035-1037`), assigned in `NewPeer` after `:1171`.
2. `coord/api.go` - the same field on `API` beside `Billing` (`:52-54`), assigned after `:133`.

Forgetting `api.go` is **invisible** in `coord run` (dev, all-in-one) and total in production, where
the api role is the only one that serves worker-facing RPCs. Hence a dedicated test (§10).

An unparseable minimum is **fatal at boot**: a coordinator that cannot say what the smallest
withdrawal is has no business accepting withdrawals. Do not call `Config.Parse()` here - that would
turn a malformed `RateGetTB`, which today only breaks a monthly report, into a coordinator that will
not boot.

## 8. Worker CLI

### The dial problem, and why both obvious answers are wrong

**Not the full peer path** (`worker/contact/service.go:297`): `revocation.OpenDBFromCfg` defaults to
`bolt://$CONFDIR/revocations.db` with revocation extensions on by default, and bbolt takes an
**exclusive flock**. The running daemon holds it, so `aioznode withdraw` would fail with a bolt
timeout on every machine that has a balance. `trust.NewPool` additionally needs - and writes to - the
worker's SQLite `info.db`.

**Not the keytool path** (`cmd/keytool/cmd_authorize.go:54`): it takes a hand-typed `--signer.address`
and `certclient.New` uses `UnverifiedClientTLSConfig()` (`certificate/certclient/client.go:31`), which
**does not pin the server's node id**. Correct for `authorize`, a phishing surface for a money RPC.

**Use a third**: trust-list resolution without the pool, then the real ID-pinning dialer. New package
`worker/payoutclient`:

| Step | Call |
|---|---|
| identity | `cfg.Identity.Load()` (as `cmd/worker/api.go:19`) |
| tls, **nil** revocation DB | `tlsopts.NewOptions(ident, sc, nil)`, `sc.UsePeerCAWhitelist = false` |
| trust cache | `trust.LoadCache(cfg.Trust.CachePath)` |
| trust list | `trust.NewList(log, cfg.Trust.Sources, cfg.Trust.Exclusions.Rules, cache)` |
| resolve | `list.FetchURLs(ctx)` - falls back to the on-disk cache when a source is unreachable |
| dial, **pinning the coord node id** | `dialer.DialNode(ctx, nodeURL, dial.Options{})` |

Coordinator selection: 0 trusted -> error naming `--worker.trust.sources`; exactly 1 -> use it and
print it; >1 without `--coord <uuid>` -> error listing all of them. **Never guess which coordinator a
withdrawal goes to.**

### Commands

```
aioznode balance  [--coord <uuid>] [--json] [--timeout 20s]
aioznode withdraw <address> [--amount <usd>] [--coord <uuid>] [--yes] [--dry-run] [--json]
```

`--amount` takes **dollars** (`12.50`), parsed `decimal.NewFromString` -> `vo.NewUSDFromDecimal`, the
same way `coord billing credit` does at `cmd/coord/billing.go:245-255`. Requiring a human to type
`12500000000000000000` is a way to lose a factor of ten. All output uses `vo.USD.Dollars()`; attousd
is a wire format, not a UI.

`balance` prints payable, pending (with the request id), available, lifetime owed/paid, the withheld
bond and how much is released, and the minimum. **No AIOZ figure and no rate line at all** - not even
an "unavailable" placeholder, which would only advertise a number the worker is not being given.

`withdraw` prompts by default and shows the address in **both encodings** - the same trick
`GetDepositAddressResponse` uses - because a mistyped bech32 that still parses produces a visibly
different hex. `--yes` skips it. **If stdin is not a TTY and `--yes` was not passed, refuse**: never
assume yes, never block forever on a pipe. `--dry-run` calls only `GetPayoutBalance`, validates
locally, prints the same block, and exits non-zero if the amount would have been refused, so it works
as a script precondition.

New: `cmd/worker/withdraw.go`, two constructors in `cmd/worker/command.go`, and two `process.Bind`
blocks in `cmd/worker/main.go` after the `setupCmd` bind at `:75`, plus extending `:77`. Bind the
existing `apiConfig` (`*worker.Config`) - it already carries `Identity`, `Trust`, `Server`. Per-leaf
binding is mandatory for the reason spelled out at `cmd/coord/main.go:292-294`.

## 9. Operator CLI

New file `cmd/coord/withdrawals.go`, following the one-shot shape of `cmd/coord/compensation.go`.

```
coord compensation withdrawals list   [--status pending] [--worker <uuid>] [--limit 100]
                                      [--output <file>] [--csv]
coord compensation withdrawals settle <id> --receipt <tx-hash>
                                      [--amount-aioz <attoaioz>] [--usd-rate <rate>]
                                      [--notes <text>] [--off-chain]
coord compensation withdrawals reject <id> --reason <text>
```

`settle` with neither `--amount-aioz` nor `--usd-rate` calls the **existing**
`resolveAiozRate(ctx, cmd, dbConn)` (`cmd/coord/compensation.go:238`) and converts with
`fx.USDToAioz`, so a withdrawal is priced exactly the way a period is - same treasury rate, same
`--rate-override`, same `--max-rate-deviation-percent` guard, same basis printed to stderr. Supplying
one flag without the other is rejected. `--receipt` is required; blank only with an explicit
`--off-chain`, because on this path a zero-AIOZ payment is indistinguishable from a deliberate
off-chain correction.

Registration after `cmd/coord/main.go:217`, and all three leaves added to the per-leaf bind loop at
`:295-307` (they all need the DB DSN; `settle` also needs `Deposit` for `spotSource`). `payoutsCSVCmd`
stays out of that loop as today. `list` renders with `text/tabwriter`, matching
`cmd/coord/billing.go:308-317`, and reuses `openOutput` (`compensation.go:515`).

## 10. Tests

**`internal/compensation`** - `ParseWithdrawal` rejects `""`/`"0"`/`"-1"`/`"abc"` and is independent
of the rate fields; `DefaultConfig().ParseWithdrawal()` succeeds; the four `WithdrawalStatus`
constants are exactly the four in the `000045` CHECK (string-set comparison, so adding one in Go
without a migration fails here).

**`coord/db`** (`coorddbtest.Run`, `DEPIN_TEST_POSTGRES=postgresql://admin:admin123@localhost:5445/hub?sslmode=disable`):

- **`TestConcurrentWithdrawalRequestsProduceExactlyOnePendingRow`** - 16 goroutines behind a start
  gate; exactly 1 success, 15 `ErrWithdrawalExists`, 1 row. **Land this with the repository, not
  after** - it is the only thing proving the index does its job.
- `TestSettleWritesThePaymentAndFlipsTheStatusAtomically`; `TestSettleFreezesTheAiozAmountAndRate`
  (`fx.AiozToUSD(amount_aioz, usd_rate)` reconciles to the USD settled);
  `TestSettleTwiceWithABlankReceiptDoesNotDoublePay` (the case `000036` does not cover);
  `TestSettleRollsBackWhenThePaymentInsertFails`; `TestSettleIsRefusedWhenOvertakenByRecordPeriod`
  (§3c step 3); `TestRejectReleasesTheReservation`; `TestRequestKeyReplayReturnsTheSameRow`;
  `TestCreateAfterSettleIsAllowed` (the partial index must not be a lifetime lock);
  `TestPendingTotalSumsOnlyPendingRows`; `TestAddressCanonicalisationRoundTrips` plus explicit
  rejections of 32-byte bech32, the zero address, and bad EIP-55;
  `TestPayableIsUnchangedByAPendingWithdrawal`.

**`coord/payout`** (fakes, no DB; style of `coord/billing/endpoint_test.go`):
`TestGetPayoutBalanceIsScopedToTheCaller`; `TestWithoutAPeerIdentityIsUnauthenticated` (not
`Unknown`); `TestAvailableSubtractsPendingRequests`; **`TestRequestWithdrawalHasNoWorkerIdField`** -
reflection over the generated request struct finds no field containing `Worker`, so the
cannot-withdraw-for-another-worker property is asserted *structurally* and cannot regress by someone
adding the field back; **`TestPayoutServiceExposesNoRateToWorkers`** - the same reflection trick over
every message in the generated package, asserting no field name contains `Aioz`, `Rate`, `Receipt`
or `Denom`. The no-rate rule is a product decision that reads as a missing feature, so it needs a
test that fails when somebody helpfully adds it back. Plus the validation table's rejections, each
asserting the store was never called.

**`cmd/worker`** - `resolveCoordinator` picks the only trusted one, refuses to guess between several,
rejects an id not in the trust list, and names `--worker.trust.sources` when there are none;
confirmation refuses on a non-TTY without `--yes`; `--dry-run` issues no request;
`--amount 12.50` is 12500000000000000000 attousd.

**`internal/testplanet`** - `TestWorkerWithdrawalEndToEnd` reuses the body of `TestCompensationEndToEnd`
up to `RecordPeriod` so the balance is genuinely earned, then asserts the request creates a pending
row, `available` drops to 0 while `payable` is unchanged, a second request is refused, settle moves
`TotalPaid` and reconciles the AIOZ, and a fresh request is refused below the minimum.
`TestWorkerCannotWithdrawForAnotherWorker`. `TestConcurrentWithdrawalsAcrossTheRPCCreateOneRow` - the
endpoint-level version, where the read-then-write TOCTOU actually lives.
**`TestPayoutServiceIsRegisteredOnTheApiRole`** - build a `coord.NewAPI` and assert the service is in
`GRPC().GetServiceInfo()`; cheap guard against the wired-in-peer.go-forgotten-in-api.go bug.

**`docs/test-plan.md`** - new rows TC-RWD-17..25 covering: balance read scoping, request creates a
pending row without moving money, below-minimum and over-available refusals, the concurrency
regression, the cross-worker negative, settle atomicity plus AIOZ reconciliation, double-settle
refusal, a withdrawal succeeding on a coordinator with no price source, and a negative row asserting
no worker-facing message carries an AIOZ amount, rate or receipt.

## 11. The `Sender` seam for phase 2

`coord/payout/sender.go`. Not `internal/compensation` (leaf, must never know about chains) and not
`coord/deposit` (money-in plus custody; its "spend the whole balance minus gas" model is exactly wrong
for a payout).

```go
type Sender interface {
    // Send returns a hash once the node accepts the transaction for BROADCAST, not
    // once it is confirmed. IdempotencyKey is the withdrawal's request key; an
    // implementation that can derive a deterministic nonce from it MUST.
    Send(ctx context.Context, req SendRequest) (SendResult, error)
}
type SendRequest struct {
    To vo.AccAddress; AmountAioz vo.AiozCoin; AmountUSD vo.USD
    USDRate decimal.Decimal; IdempotencyKey vo.UUID; WithdrawalID int64
}
type SendResult struct{ TxHash string; BroadcastAt time.Time }
```

Phase 1 holds a nil `Sender` and takes the manual branch. The interface is defined now because the
shape of a broadcast decides the shape of the state machine: retro-fitting "in flight, hash unknown"
into a settle path that assumes the receipt is known is how a network double-pays. Phase 2 orders it
*mark -> broadcast -> record*, and on an ambiguous Send error leaves the row `settling` for a human -
"we might have sent it" must never auto-release a reservation.

Phase 2's total delta: `'settling'` in the CHECK, the one-pending index and reserved sum widened to
`IN ('pending','settling')`, and two nullable columns. Everything in §1-§10 is unchanged.

---

## Files touched

| Area | Path |
|---|---|
| Migration | `coord/db/migrations/000045_worker_withdrawals.{up,down}.sql` |
| Leaf types + config | `internal/compensation/withdrawal.go` (new), `config.go`, `rates.go`, `db.go` (doc xref) |
| Repository | `coord/db/withdrawal_repo.go` (new), `compensation_repo.go`, `errors.go` |
| Proto | `pkg/pb/coord/payout/v1/payout.proto` (new) + `make proto` |
| Endpoint | `coord/payout/{endpoint,service,sender}.go` + `README.md` (new package) |
| Wiring | `coord/peer.go`, `coord/api.go` |
| Worker client + CLI | `worker/payoutclient/client.go` (new), `cmd/worker/withdraw.go` (new), `cmd/worker/command.go`, `cmd/worker/main.go` |
| Operator CLI | `cmd/coord/withdrawals.go` (new), `cmd/coord/main.go`, `cmd/coord/compensation.go` (the two guards in §5) |
| Docs | `internal/compensation/README.md`, `docs/test-plan.md`, `docs/RELEASE.md` |

## Traps found during exploration

- **`Makefile.migration:15`** sets `COORD_DB_PATH ?= ./coord/infrastructure/db/migrations`, which does
  not exist; migrations live in `./coord/db/migrations`. Create `000045` by hand or fix the variable.
- **`runCmd` is registered but never bound** (`cmd/worker/main.go:38,77`), and `cmdRun` is a
  `return nil` stub. Anyone modelling a new subcommand on it gets a zero-valued `worker.Config`, which
  presents as "identity file not found" and looks like user error. **Model on `apiCmd`.**
- **`WorkerConfigCmd()` / `SetStorageConfig()`** (`cmd/worker/command.go:44-69`) are dead - never
  registered, and `setupStorage` hand-edits `config.yaml` under a key that does not exist in
  `worker.Config`. Do not extend it.
- **`os.RemoveAll` marked "TODO: remove after testing"** at `cmd/worker/api.go:26-31,42-47` sits in the
  file a new worker subcommand lands beside. Do not copy it.
- **`internal/compensation/README.md:150-151`** is now half-wrong: the *payment* needs no schema
  change, the *request queue* does.

## Sequencing

1. Proto + `make proto` - independent, lets both sides compile.
2. `internal/compensation/withdrawal.go` + `MinWithdrawalUSD` / `ParseWithdrawal` + unit tests.
3. Migration `000045`, then `coord/db/withdrawal_repo.go` **with the concurrency test**.
4. `coord/payout/endpoint.go` + fakes-based tests.
5. Wiring in `coord/peer.go` **and** `coord/api.go`, plus the api-role registration test.
6. `cmd/coord/withdrawals.go` + registration and per-leaf bind.
7. `worker/payoutclient` + `cmd/worker/withdraw.go` + binds.
8. `internal/testplanet/withdrawal_test.go`.
9. Docs.

## Verification

1. `go build ./...`, `go vet ./...`
2. `go test ./internal/compensation/... ./coord/payout/...`
3. `DEPIN_TEST_POSTGRES=... go test ./coord/db/... -run Withdrawal -count=5` - the concurrency test
   must pass repeatedly and under `-race`.
4. `DEPIN_TEST_POSTGRES=... go test ./internal/testplanet/ -run 'Withdrawal|Payout'` (needs the local
   `replace ../go-sdk` in `go.mod`).
5. **Manual run against a throwaway database**, the way the Phase-2 double-recording bug was caught
   when every automated test passed: seed a worker with a priced period, run `aioznode balance`, file
   a withdrawal, confirm `payable` is unchanged and `available` dropped, try a second request, settle
   it, run settle again, and cross-check in SQL that `SUM(amount_aioz) * usd_rate = SUM(amount)` and
   that exactly one `worker_payments` row carries that `withdrawal_id`.

---

## Outcome (2026-08-10)

Implemented in full on `feat/reward-flows`, uncommitted. All nine sequencing steps done.

Three bugs the tests caught during implementation, all in newly written code:

1. **The pending-count and the balance were read in separate statements.** Under READ
   COMMITTED each statement takes its own snapshot, so a concurrent request could be
   visible to one and not the other - and the loser of a race was told "insufficient
   balance" instead of "you already have a request open". Fixed by returning both from
   the single availability CTE. The 16-goroutine test went 15 wrong -> 3 wrong -> 0.
2. **The EIP-55 check used `strings.EqualFold`.** The checksum *is* the
   capitalisation, so a case-insensitive comparison validated nothing. Fixed to a
   case-sensitive compare. (The test's own "valid" constant also turned out not to be
   checksum-valid.)
3. **A pre-existing broken build.** Merge `308445b` (`develop` -> `feat/reward-flows`)
   kept `type DepositAddress struct` while taking develop's `[]Address` usages, so
   `go build ./...` failed at `coord/deposit/service.go:64`. Renamed to `Address`.

Deviations from the plan as approved:

- `contact.Worker` gained a `DisqualifiedAt *time.Time` field. The column existed
  (migration 000027) but was not on the Go model, so the disqualification check had
  nothing to read. Follows the `Exiting` field's precedent.
- The worker-visible `Withdrawal` message returns the address in **both** encodings
  rather than one, matching `GetDepositAddressResponse` - a mistyped bech32 that still
  parses produces a visibly different hex.
- `CancelWithdrawal` was added (the plan listed it as recommended). Without it, one
  pending request at a time strands a worker that typed a valid-but-wrong address.
- The testplanet e2e seeds `owed` through `RecordPeriod` instead of uploading and
  tallying. What turns usage into `owed` is already proved by
  `TestCompensationEndToEnd`; these tests cover everything downstream of a balance.

Known loose ends, deliberately not addressed: `Makefile.migration` still points at a
non-existent migrations directory; `runCmd` is still registered without a config bind
and `cmdRun` is still a stub; `WorkerConfigCmd`/`SetStorageConfig` are still dead code;
the `os.RemoveAll` "TODO: remove after testing" in `cmd/worker/api.go` is still there.

---

## Follow-up: automatic settlement of small withdrawals (2026-08-10)

Phase 2, scoped to small amounts. A request at or below
`--payout.auto-settle-max-usd` (default $5) is broadcast and recorded by a chore
without an operator; anything larger still waits for a human. Risk tiering, not
convenience.

New: migration `000046` (the `settling` state, `broadcast_tx_hash`/`broadcast_at`,
and the widened reservation index), `coord/payout/autosettle.go`,
`coord/payout/ethsender.go`, `coord/payout/config.go`, and wiring on the core role
plus the all-in-one `run` role.

**The ordering is the contract**: mark settling -> broadcast -> record hash -> write
payment, each durable before the next. Broadcasting cannot be undone and cannot be
reliably observed, so the row must already say "a transfer may be in flight" before
anything leaves. A settling row is never reopened on an ambiguous failure -
`ReturnToPending` refuses once a hash exists, and the settler treats that refusal as
"leave it for a human". Leaving a row stuck costs an operator five minutes; reopening
one whose transaction landed pays the worker twice.

**The trap 000045 predicted, now live**: a settling row still holds its reservation.
`pendingStatuses`, `idx_worker_withdrawals_one_pending` and
`WithdrawalStatus.HoldsReservation()` are three copies of one rule; if they disagree a
worker can withdraw the same money while the first transfer is in the mempool.

Decisions made rather than asked:
- The treasury key is a **file path**, not a config value, and the loader refuses a
  group- or world-readable file. A private key in `config.yaml` ends up in backups,
  logs and git.
- The chore carries an `enabled` flag (off by default) on top of being inert without
  a key. The flag says an operator meant to do this; the key is what makes it
  possible.
- Singleton, on the core role. Two settlers would bid for the same treasury nonce,
  and the per-row status guards do not protect against that.

Rails deliberately not built (the user picked only the per-worker cooldown): a
per-run total cap and a dry-run mode. Both are small additions later.
