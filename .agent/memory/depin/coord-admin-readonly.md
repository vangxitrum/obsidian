---
type: decision
tags: [depin, coord, admin-ui, security, read-only]
created: 2026-09-03
agent: main
---

The coord admin back-office is now **structurally incapable of writing the
coordinator's database**. Branch `feat/coord-ui`, uncommitted as of 2026-09-03.

## What was there before

Read-only was already the DEFAULT (`admin.database.coord-write-url` empty ->
writer interfaces nil -> mutating endpoints answered "not configured"), plus
`ReadOnlyGuard` stamping `default_transaction_read_only=on`. But it was a
CONFIGURATION, and an operator could turn writes on.

## What was deleted

- Routes/handlers: `POST /admin/workers/:id/disqualify`, `.../reinstate`,
  `POST /admin/billing/credit`, `POST /admin/withdrawals/:id/reject`.
- Interfaces `Disqualifier`, `BillingWriter`, `WithdrawalWriter` and the
  `Operations` fields `Disqualify` / `BillingWrite` / `SettleW`.
- Permission bits `PermWorkerDisqualify`, `PermWorkerReinstate`,
  `PermBillingCredit`, `PermWithdrawalReject` (+ their names and role grants).
- Config `admin.database.coord-write-url`, `adminConns.write`, and the
  `AdminReadOnly` testplanet knob (there is only one shape now).
- The UI buttons/dialogs in WorkersView, BillingView, WithdrawalsView and the
  four `api.*` client methods.

## Consequences worth knowing

- `AuthSupport` is now **identical to AuthViewer**, and `AuthFinance`'s extra
  bits gate NO endpoint - they were already CLI-only vocabulary (verified:
  `PermBillingRecordPeriod`, `PermWithdrawalSettle`, `PermDepositSweep`,
  `PermPeriodCloseView`, `PermDepositView` have zero non-test references and did
  before this change too).
- CLI replacements: `coord billing credit`, `coord compensation withdrawals
  reject|settle`, and `coord worker disqualify|reinstate` - the last one was
  ADDED on 2026-09-04 in `cmd/coord/worker.go`, because the removed admin
  endpoint had been the only caller of `overlay.DB.DisqualifyWorker` and killing
  it left no way to disqualify a worker at all. `--reason` is required (>= 8
  chars, mirroring the endpoint's `requireReason`); both commands are no-ops on
  a worker already in the target state. (`DQWorkersLastSeenBefore` is still
  caller-less, and was before any of this.)
- Removed routes return **404, not 403** - the SPA fallback in `static.go` sends
  a plain 404 for anything under `/api` that reached it.

`TestAdminCannotWriteTheCoordinator` (was `TestAdminReadOnlyDeployment`) asserts
this as a SUPER ADMIN on purpose: a narrow role would pass on a 403 and prove
nothing.

Related: [[coord-admin-ui]], [[period-close-dashboard-heartbeat]].

## Trap: concurrent edits in this worktree

Commit `758f5eb` shipped BROKEN (`undefined: strconv` in handlers_ops.go).
Another session added a per-category earnings breakdown to `handleListPaystubs`
that uses `strconv.FormatInt`, while this change removed the `strconv` import -
correct at the time, since its only user was `handleRejectWithdrawal`. Both were
committed together. Fixed on 2026-09-04 by restoring the import.

Lesson: this worktree can have more than one agent editing the same file. A
green `go build` is only evidence about the tree AT THAT MOMENT - re-run it
immediately before committing, and be suspicious of an import removal when the
file's line count has changed underneath you.
