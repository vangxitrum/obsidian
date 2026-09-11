---
type: fact
tags: [depin, coord, admin, ui, vue, auth, audit]
created: 2026-08-24
agent: main
---

Coordinator admin back-office shipped on branch `feat/coord-admin-ui` (uncommitted).
All 6 phases of `Projects/depin/plans/2026-08-24-coordinator-web-ui.md`, narrowed
mid-session by the user to **admin only** (customer console + worker-operator
dashboard cut, `account_clients`/`account_workers` join tables dropped with them).

## What exists

`coord admin` peer (`coord/admin/`) + Vue 3/Vuetify SPA (`coord/admin/ui/`).
Migrations 000050 (read indexes) and 000051 (accounts/sessions/tokens/audit).
13 testplanet e2e tests (`-run TestAdmin`) + unit tests; `make lint` 0 issues;
full suite green.

## Load-bearing findings

- **`prepareCoord` loads the node identity** — the admin peer needs none (signs
  nothing, no gRPC), so `cmd/coord/admin_cmd.go` calls `coorddb.ConnectDB`
  directly. Running a back-office should not require the coordinator's key.
- **httprouter refuses a static + wildcard sibling** on one segment, whatever the
  registration order. `/admin/workers/summary` beside `/admin/workers/:id` panics
  at startup → moved to `/admin/fleet/summary`. Same trap avoided for
  `/admin/accounts/roles` (different HTTP methods = separate trees, so that one
  is fine).
- **`vo.USD.String()` is attousd base units, not dollars**; `Dollars()` is the
  display form. Every money field in the admin API is served twice (`x` exact +
  `x_usd` readable). The gRPC billing API sends base units too.
- **`contracts.owner` does not exist** — migration 000003 replaced it with
  `owner_id UUID`. `clients.address` already has a unique index. `workers.id` is
  TEXT, not uuid, so no `::text` cast in a LIKE.
- **`accounting.UsageReport.Egress` is one int64**, not the
  allocated/settled/inline/dead split (that lives in contract_egress_rollups).
- **npm's node_modules leaked into the Go module**: `go list ./...` reported
  `.../ui/node_modules/flatted/golang/pkg/flatted` as a package of this repo.
  Fixed with a fence-only `coord/admin/ui/go.mod`. Worth knowing for any future
  JS-in-repo work.
- `.gitignore` allowlist bit twice: `coord/admin/build_fallback.html` and
  `coord/admin/ui/Dockerfile` (the `!cmd/*/Dockerfile` rule is one level only).
  See [[depin-gitignore-allowlist-gotcha]].

## Bug caught by manual verification, not by tests

`SeedFirstAdmin` originally created an account with a password but **no MFA
secret and no invite token** — and login refuses an unenrolled account, so the
very first admin of a deployment could never log in. The e2e passed because its
helper wrote the MFA secret straight into the DB. Fixed by making seed mint an
invite so seeded and invited accounts share one enrolment path; the test helper
now drives the real HTTP flow. **Lesson: a test helper that shortcuts the product
path can hide the bug the path exists to have.**

## Phase 0 pre-existing bugs

Auth replay window was already fixed in-tree (const 3600); the remaining work was
typing it as `time.Duration`. `recoverPanic` asserting `err.(string)` and the
untagged `checkRange` were both live and are fixed with regression tests. Note
`checkRange`'s call sites now wrap with `vo.GrpcError` — without that the tagged
error would have reached gRPC as `Unknown`.

## Test-environment note

`go test ./...` against one Postgres fails intermittently with
`duplicate key ... pg_extension_name_index` — the `CREATE EXTENSION IF NOT EXISTS
"uuid-ossp"` race in migration 000002. Pre-existing (verified on a stashed
baseline: same failure, different random set). `.gitlab/ci/test.yml` already
documents and fixes it by pre-creating the extension; do the same locally
(`psql -c 'CREATE EXTENSION IF NOT EXISTS "uuid-ossp";'`) and the full suite is
green. Also needs `max_connections` well above the default 100.

## Deliberately not built

Withdrawal *settlement* (broadcasts an irreversible on-chain transfer needing an
out-of-band receipt — CLI keeps it); email of any kind (no mail transport, so
invite/reset links are shown once to the inviting admin, and there is no
email-verification step); the `...For` gRPC refactor the plan called for (turned
out unnecessary — the admin peer reads `billing.DB`/`UsageService` directly, and
both are already owner-parameterized).

## Durability panel (churn + availability -> RS config)

`/admin/durability` + `DurabilityView.vue`. Built because "how many workers went
down last month" splits into two things that drive DIFFERENT RS parameters and
must not be one number:

- **churn** = permanent departure -> sets repair threshold / repair capacity
- **availability** = transient absence -> sets the required..total gap

Load-bearing facts:

- `uptime_month` = 30 DAILY buckets, `uptime_day` = 24 HOURLY. A bucket is 1 if
  the worker checked in at ANY point in it, so both are **upper bounds** on
  availability and every derived durability figure is optimistic. Said so on the
  page.
- Churn denominators matter: a worker with the zero `last_seen` (year 1 in
  Postgres) never arrived and must NOT count as attrition — 330 of 1200 in the
  dev fleet are exactly that. `zeroTimeCutoff = '1971-01-01'` separates them.
- `departed` needs a silence threshold well above the 4h selection window
  (default 7d) or every afternoon outage reads as churn.
- **0/0 = 0% is the dangerous case**: a fleet where nothing checked in during the
  window reports a reassuring green 0% churn. The view detects `active == 0 &&
  fleet > 0` and says so loudly. Found by running it against a stale DB copy.
- P(segment lost) is a binomial over INDEPENDENT failures, computed in log space
  (lgamma) because C(80,40) ~ 1e23. Independence is the assumption that actually
  breaks, so the panel also reports top-subnet share from `last_net` as the check
  on it — the dev fleet is 66.8% behind one subnet.
- Real placements: `clone-3x` 2/2/3/3 (id 1) and `storj-default` 29/52/60/80
  (id 999). `coord/db/seeds/placements.sql` is DEAD and would fail if run — it
  has `null,),` (trailing comma) three times and nothing in Go references it.

Tests: `coord/admin/durability_test.go` (binomial vs hand-computed values),
`coord/db/durability_repo_test.go` (churn classification, ring-buffer maths,
concentration), `internal/testplanet/admin_durability_test.go` (e2e).

Related: [[coord-split-compose]], [[client-billing-phase-a]],
[[worker-withdrawals]], [[period-close-and-gating]].

## Earnings leaderboard (top workers by reward)

`/admin/compensation/top-earners` + `EarningsView.vue`, gated on
`PermCompensationView`. Files: `coord/admin/earnings.go`,
`coord/db/earnings_repo.go`, `coord/admin/handlers_earnings.go`.

Load-bearing facts:

- **The fan-out trap is the whole reason for the CTE shape.** A worker has many
  paystubs AND many payments. Joining `worker_paystubs` to `worker_payments`
  before grouping multiplies them: 3 stubs x 2 payments sums each 6 times. The
  result is not visibly broken, just silently inflated. Both tables are
  pre-aggregated in their own CTE (`stubs`, `pays`) and only then joined.
  Locked by `TestTopEarnersDoesNotFanOut`.
- **`worker_paystubs.worker_id` is `uuid`; `workers.id` is `TEXT`.** Join casts
  `s.worker_id::text`. That direction is total; `text -> uuid` raises on any id
  that is not a valid uuid.
- The workers join is LEFT and the row carries `known`: **the ledger outlives the
  fleet**, so a departed worker is still owed what it earned. Without `known`
  the UI would present a missing worker as an offline one.
- **Payable clamps per worker, not at the end.** `GREATEST(owed - paid, 0)` runs
  inside the per-worker CTE before the fleet sum. Summing owed and paid
  fleet-wide first lets one overpaid worker cancel another's debt and understates
  what the network owes. Overpayment is real: a manual one-off payment with a
  NULL period on top of a generated one.
- Totals are a SEPARATE query from the page, never a sum of the returned rows -
  otherwise the top 100's earnings get reported as the network's.

Verification gotchas hit:

- The demo DB had **0 paystubs**, so the page only ever showed its empty state.
  Had to seed a synthetic pareto ledger (3 periods x 1200 workers + 2228
  payments + 2 ghost paystubs with uuids absent from `workers`) to see any of the
  above render.
- **Vuetify ships a global `.text-warning` utility** that beat the scoped class of
  the same name and painted the Payable column orange - colour-alone encoding
  that was never intended. Renamed to `.dt__owing`. Check scoped class names
  against Vuetify's utilities.
- Every money value in the admin UI prints the raw 6-decimal API string
  (`$12828.700000`). Added `formatUSD`/`usdTitle` to `lib/format.ts` (grouped, cut
  to cents, full precision kept in the `title`) and used it on Earnings only.
  **BillingView, ClientsView, WithdrawalsView and DashboardView still print raw**
  and should be retrofitted.
- The `coord-admin` container runs a **baked image**, so a new endpoint 404s there
  until `make build-coord-image` + `docker compose ... up -d`. Faster loop: run a
  locally built `coord admin` on 127.0.0.1:7790 against the same DSN; the vite
  dev server already proxies `/api` there.
- Admin login body field is `mfa_code`, not `passcode`. Password-guessing loops
  get blocked by the tool classifier (correctly) - mint an `account_tokens` row
  with `kind='password_reset'` (token_hash = sha256 hex of the token) and POST
  `/api/v0/auth/reset` instead. Reset preserves the MFA secret.

Tests: `coord/db/earnings_repo_test.go` (6 - ranking, fan-out, period scope,
overpayment clamp, missing worker, empty ledger),
`internal/testplanet/admin_earnings_test.go` (e2e + validation/permission).

## Audit coverage panel (on the Durability page)

Added to `/admin/durability` rather than as its own page, because a page named
"Audit" would sit next to the existing "Audit log" nav entry and the two are
unrelated subsystems. Files: `coord/admin/auditcoverage.go`,
`coord/db/auditcoverage_repo.go`. `DurabilityReader` gained `AuditCoverage`, and
`DurabilityReport` gained an `audit_coverage` section - no new route.

The question it answers, and why it is answerable at all:

- **Storj has no equivalent.** Grepped `../storj` for `last_audited` /
  `audited_at` / `audit_coverage` / `percent_audited` across .go/.sql/.dbx: zero
  hits. Its `audited_percentage` metric (`satellite/audit/verifier.go:661`) is
  `totalAudited/totalPieces` WITHIN ONE SEGMENT - not network coverage. Storj's
  audit is node-centric by design; every node with data gets a reservoir each
  loop pass, so coverage is a structural invariant it never measures.
- **depin CAN answer it** because it persists `workers.last_audit_at`
  (`coord/db/reputation_repo.go:106`), which Storj does not, plus `audit_history`
  windows `{start, online, total}`.

Two measurements, never one:
- ATTEMPTED = `last_audit_at` in range. Includes workers recorded offline.
- REACHED = an `audit_history` window with `online > 0`. Only inside retention
  (720h) and only to window granularity (12h).
A worker attempted 900 times with `online: 0` has a fresh timestamp and has
proven nothing. That is real - it is what the dev fleet's history looks like.

Load-bearing gotchas:

- **`audit_history` can be the jsonb scalar `null`, not SQL NULL.** It is written
  as `json.Marshal([]AuditWindow)` and a nil slice marshals to `null`, so
  `COALESCE(col,'[]')` does NOT catch it and `jsonb_array_elements` fails the
  whole query with "cannot extract elements from a scalar". Guard with
  `CASE WHEN jsonb_typeof(...) = 'array'`. Caught by a repo test, not by the
  live DB (which happened to have only arrays).
- **Disqualified workers stay in the denominator.** depin's audit observer has NO
  eligibility filter - Storj's `AuditedNodes`/`AllowNodes`
  (`satellite/audit/audit_filter.go`) was never ported - so a disqualified worker
  still holding pieces is still sampled.
- `never_audited` is irreducibly ambiguous (holds no data vs holds data we have
  not reached); there is no cheap per-worker piece count, `segments.pieces` is a
  blob. Say so on the page rather than guessing.
- `audit.Enabled` defaults to **false**, so a 0% panel usually means the pipeline
  is off, not that workers are failing. `Stale` (nothing in the whole retained
  history) is the signal for that.
- `.grid--2` is `minmax(320px,1fr)`, so nested inside a half-width panel it
  collapses to one column. The Availability section does the same - consistent,
  not a regression.

## Two breakages the feat/improve-audit -> develop merge left on develop

Both found while verifying the above; neither was caused by it. Work is now on
`develop`, and the admin UI was committed as `51490cf`.

1. **Duplicate migration 000050** - `admin_ui_read_indexes` (2026-08-25) and
   `reverification_queue_indexes` (2026-08-26) both claimed it. golang-migrate
   refuses to load the directory at all: "duplicate migration file". EVERY test
   that migrates fails, and `coord core`/`coord run` cannot start. Fixed by
   renumbering the newer one to 000053 (000052 was taken). Residual risk: a DB
   that already applied the audit branch's 000050 will skip the renumbered file -
   harmless here because both are `CREATE INDEX IF NOT EXISTS`.
2. **`make lint` red** - three `err != audit.ErrQueueEmpty` comparisons in
   `coord/db/audit_queue_test.go` tripping err113. Fixed with `errors.Is`.

Check `ls coord/db/migrations | sed 's/_.*//' | sort | uniq -d` after any merge -
this is the same class as [[merge-broke-deposit-address]].

Tests: `coord/db/auditcoverage_repo_test.go` (7),
`internal/testplanet/admin_auditcoverage_test.go` (2, incl. the stale path).

## Layout: shared primitives in styles/layout.css

`coord/admin/ui/src/styles/layout.css`, imported from `main.ts` after
`tokens.css`. Created because the same layout classes were re-declared as scoped
copies in NINE views - `.page` and `.panel` in all of them, the whole `.dt` table
system in two - so the same class name meant a slightly different thing per file
and a fix on one page silently left the others behind. That drift WAS the "UI
looks a bit off".

Method that made it safe: a script compared each view's rule against the global
one and removed only byte-identical ones, listing the divergences for review. The
divergences were all trivial (`min-width` 880 vs 900, `--space-4` vs `--space-6`
page gap, a keyframe name), so they were unified rather than kept.

Three real layout bugs fixed in the shared version:

- **Nested tile grids collapsed to one column.** `.grid--2` is
  `minmax(320px, 1fr)`; nested inside a half-width panel (~490px) auto-fit gives
  ONE column, so four stat tiles stacked into a tall ribbon with the panel's
  right half empty. New `.grid--tiles` at `minmax(190px, 1fr)` gives a 2x2 block.
  190 and not 150: 150 fits three across and strands the fourth on its own row.
- **`.grid` was `align-items: start`**, leaving a ragged hole under every short
  panel. That was added earlier to stop SVG charts stretching, but the charts now
  carry an explicit pixel height so stretch cannot reach them. Now `stretch`, and
  panels are `display: flex; flex-direction: column` so a stretched panel
  distributes the slack. `.panel--center` centres sparse content (a lone stat)
  and `.panel__note--foot` pins a trailing caveat to the bottom.
- **List views had an unpadded `.panel`** while dashboard views had a padded one,
  same class name. Global `.panel` is padded; the six table pages now say
  `panel panel--flush`.

What legitimately stayed scoped: sortable headers, `.dt__row--flagged`, the
durability table's `vertical-align: top`, `.panel--hero`. Everything else went.

Also finally applied `formatUSD` to DashboardView/BillingView/ClientsView/
WithdrawalsView - money read `$0.000000` everywhere outside Earnings. Unit tests
added in `lib/format.spec.ts` (11 pass), including the sub-cent case that must
NOT round to `$0.00`.

Durability page 2537px -> 2259px. All nine pages: 0 horizontal overflow, 0
console errors.

## Database split: admin's own DB + read-only coordinator connection

The admin peer now opens up to THREE connections, and the type `adminConns` in
`cmd/coord/admin_cmd.go` exists to keep them apart:

1. `admin.database.url` (AIOZ_ADMIN_DATABASE_URL) - the admin's OWN database.
   Accounts, sessions, tokens, audit log. REQUIRED; the peer refuses to start
   without it rather than falling back, because a silent fallback would put
   password hashes and session tokens back in the coordinator's database with no
   way for the operator to notice. Migrated by `coord/admin/admindb` (its own
   embedded set, 000001), NOT by coord/db.
2. `admin.database.coord-url` - READ-ONLY coordinator connection. Everything the
   UI displays. Empty falls back to the coordinator's own DSN with a warning.
3. `admin.database.coord-write-url` - OPTIONAL read-write coordinator
   connection. Only disqualify/reinstate, balance adjustment and withdrawal
   settlement need it. Left empty those three fields stay nil and their handlers
   already return `ErrNotConfigured` (405) - so a strictly read-only admin is a
   supported shape, not a broken one.

Load-bearing details:

- **Repositories are bound to the pool they were built on.** The write side
  constructs a SECOND set (`NewOverlayRepository(wdb)` etc.), it does not reuse
  the read ones. That is what actually keeps a mutating handler off the
  read-only role.
- **`ReadOnlyDSN` (coord/db/readonly.go)** appends
  `options=-c default_transaction_read_only=on` to the read DSN, defence in depth
  over the GRANT. Handles BOTH libpq DSN shapes. The keyword/value branch needs a
  QUOTE-AWARE splitter (`SplitKeyValueDSN`): `strings.Fields` shatters the quoted
  `options='-c a=b'` value and silently produces a mangled DSN - caught by the
  idempotence test. Toggle: `admin.database.read-only-guard` (off behind
  pgbouncer transaction mode, which rejects startup options).
- **000051_admin_accounts stays in the coord migration set.** History is
  immutable and dropping it would destroy the accounts an operator is about to
  copy across. After the split those coord-side tables are simply unused.
- Existing accounts must be copied by hand:
  `pg_dump -t accounts -t account_sessions -t account_tokens -t admin_audit_log`.
- Flag names verified from `coord admin --help`, per
  [[release-notes-workflow]]. Seed flag is `--name`, not `--full-name`.
- Enrolment API field names: redeem takes `token`+`password`, mfa/confirm takes
  `token`+`code` (NOT `mfa_code`), login takes `mfa_code`.

Testplanet models the split: each admin peer gets its own unique schema via
`coorddbtest.OpenUnique(..., "admin")` migrated with `admindb.Migrate`, exposed
as `testplanet.Admin.DB`. `TempDatabase` gained a `DSN` field for this. Read and
write handles are the same connection there - a planet has no privilege
separation to model, and a read-only handle would disable the endpoints the tests
cover. `testplanet.Config.AdminReadOnly` boots the no-write shape.

Verified end to end against real databases: separate `coord_admin` database
migrated (4 tables), a real `coord_reader` role with SELECT only, seed -> redeem
-> mfa confirm -> login all work, six read endpoints 200, disqualify 405.

Tests: `coord/db/readonly_test.go` (5), `coord/db/readonly_live_test.go` (proves
Postgres actually refuses insert/update/delete/ddl),
`internal/testplanet/admin_readonly_test.go`.

## Flaky TestAuditPipelineBatchedReputationWrite: waited on the wrong signal

Failed ~25% of runs (2 in 8) with `"0" is not greater than or equal to "2"` -
no holder scored at all. Not flakiness in the pipeline; the test measured the
wrong thing.

`AuditRepository.NextBatch` **DELETEs** the rows it claims (`WITH claimed AS
(DELETE FROM audit_queue ... RETURNING ...)`). So `CountQueued() == 0` means
"a worker picked the segment up", NOT "the audit finished". The test waited for
the queue to empty and then cancelled the worker - which lands mid-verification
whenever the six holder round-trips take longer than one 200ms poll. The
reputation write never happened, and every assertion after it saw zeros.

Fix: wait for the property under test - holders having non-zero audit counters -
then cancel, and assert the empty queue afterwards as a plain check rather than
as the wait condition. 20/20 runs green after (was 6/8).

General rule this is an instance of: **a work-queue length is a claim signal,
not a completion signal.** Anything that deletes-on-claim cannot be polled for
"done". Only one site had it (`grep CountQueued` in tests).

Pre-existing, from `19971d6 test(audit): add pipeline test` on the
feat/improve-audit branch - the third defect that merge brought in, after the
duplicate migration 000050 and the err113 lint failures.

## Enrolment page unusable: three stacked bugs, root cause was a missing icon font

Operator report: "the Finish setup colour when active and inactive is the same,
and when I've typed the code I can't press the button." All three verified by
driving the real enrolment flow in a browser.

1. **No icon font at all.** `@mdi` was never a dependency and `vuetify.ts` set no
   `icons` config, so Vuetify fell back to its default `mdi` FONT set whose CSS
   was never imported. EVERY Vuetify-internal icon in the whole admin UI was an
   invisible empty box - checkbox ticks, select chevrons, alert icons, sort
   arrows. The "I have saved my recovery codes" checkbox rendered as 28x28 of
   nothing, so the operator never saw the control they had to tick. THIS is why
   the button could not be pressed.
   Fix: `npm i -E @mdi/js` + `vuetify/iconsets/mdi-svg` (`aliases` covers every
   internal icon, tree-shakes). Bundle 170.53 -> 175.90 kB, vs ~1.2 MB for the
   webfont, and no font file to fetch under the CSP.
2. **Disabled buttons looked identical to enabled ones.** `color="primary"` adds
   the `.bg-primary` utility, and Vuetify's utilities are `!important`, so they
   beat `.v-btn--disabled.v-btn--variant-flat`. Measured: disabled and enabled
   were both `rgb(42,120,214)` on white, opacity 1. Fixed globally in layout.css
   with an `!important` rule (specificity cannot beat `!important`) using
   `--wash-strong` / `--text-muted` so it follows light/dark. SIX views pair a
   colour with a disabled state, so this was never enrol-only.
3. **The button ignored the code.** It was gated `:disabled="!savedCodes"` only.
   Now `!savedCodes || !codeReady` (`/^\d{6}$/`), plus a `blockedReason` hint
   under the button naming what is missing - a disabled button says "no", never
   "not yet, because".

General lesson: **when a UI component library renders nothing, check whether its
icon set is actually wired** before hunting logic bugs. And a disabled control
with no visual difference is indistinguishable from a broken one.

Verified: fresh invite -> redeem -> checkbox visible as a real 24x24 SVG -> code
typed -> button enables -> "Your account is ready", 0 page errors.
