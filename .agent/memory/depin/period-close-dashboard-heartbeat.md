---
type: decision
tags: [depin, coord, admin-ui, periodclose, billing, schema]
created: 2026-09-03
agent: main
---

The coord admin dashboard now has a **Billing periods** section: last client usage
charge, last worker reward payout, and a countdown to the next close. Branch
`feat/coord-ui`, uncommitted as of 2026-09-03.

## The problem that shaped the design

`coord/periodclose` closes both halves of a month (client invoices, then worker
paystubs) in ONE chore. The admin console already showed only
`OpsStats.last_closed_period` - a string plus a hand-written note.

"When is the next close" was **not answerable from the admin side at all**:

- the admin peer is a SEPARATE PROCESS (`cmd/coord/admin_cmd.go`) that reads the
  coordinator database directly and holds no `periodclose.Config`, so it knows
  neither the grace window (72h) nor the cycle interval (1h);
- `sync2.Cycle` (`pkg/common/sync2/cycle.go`) exposes **no** `NextRun()`/`LastRun()`
  accessor - only an unexported `*time.Ticker`. Nothing in the repo does.

Rejected: duplicating the schedule knobs into the admin binary's config. Two values
that drift, and a dashboard confidently counting down to the wrong moment is worse
than no countdown at all.

## What was built

1. **Migration 000054 `period_close_status`** - singleton heartbeat row (same
   `id BOOLEAN PK CHECK(id)` shape as `period_close_floor`): `last_run_at`,
   `next_due_period`, `next_due_at`, `grace_seconds`, `interval_seconds`.
2. `periodclose.NextDue(now, grace, floor, closed)` - pure, in `schedule.go`.
   Deliberately NOT built on `DuePeriods`: that applies the backfill bound, which is
   a policy about how much work one pass takes on, not about what is outstanding.
3. `StateDB.RecordRun`/`RunStatus` + repo impl; `Service.RunOnce` split into
   `closeDue` + `recordRun`, heartbeat written on EVERY pass (including the
   floor-establishing pass that closes nothing, and failed passes).
4. Admin: `PeriodCloseReader.LastClosures` (DISTINCT ON half, newest per half) and
   `.Status`; `Dashboard.Close` gated on `billing:view` (it reports amounts, unlike
   `OpsStats` under `system:view`). No new route, no new permission bit.
5. UI: three StatTiles in `DashboardView.vue`, `formatDuration` added to
   `src/lib/format.ts`.

## Two conventions that are load-bearing

- **`next_due_at` null means "due NOW, waiting on the next tick"**, not "unknown".
  Writing `now()` there gives the console a countdown target that moves on every
  refresh - reads as a stuck clock.
- **empty `next_due_period` means nothing outstanding at all** - a different fact
  from "due now", and it must not render the same.

## Other notes

- The heartbeat is a display fact: `RecordRun` failing is logged and never fails a
  pass that has already written invoices. `recordRun` also early-returns on
  `ctx.Err() != nil` so shutdown does not log a warning per stop.
- The two halves are reported INDEPENDENTLY (newest client close and newest worker
  close can be different months). `LastClosedPeriod` still requires both, and that
  reading is unchanged - a period billed to clients but not yet paid to workers is
  exactly the state the panel exists to show.
- `period.Period` gained `IsZero()`.
- e2e proof is `TestAdminDashboardReportsThePeriodClose` in
  `internal/testplanet/admin_periodclose_test.go` - it goes over HTTP against a real
  admin peer, because a heartbeat the chore writes and the admin peer cannot read
  would pass every unit test in the change.

Related: [[period-close-and-gating]], [[coord-admin-ui]],
[[billing-settlement-audit-2026-08-20]].
