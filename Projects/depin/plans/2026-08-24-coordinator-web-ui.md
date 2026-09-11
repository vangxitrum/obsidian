# Coordinator Web UI

## Context

The depin coordinator has no web interface. Every operation is either a gRPC
call from the Go SDK/CLI or a `coord <subcommand>` run by hand on the server.
Clients cannot see their own contracts, files, usage, balance, invoices or
deposits without writing code. Operators run billing, compensation,
withdrawals, deposits and period-close entirely from the CLI. Worker operators
have no view of their node's reputation or earnings.

Storj solves this with four Vue 3 UIs (see
`/home/tuan/work/depin-workspace/storj/ui.md`). This plan ports that model,
adapted to depin's two structural differences: depin has **no user accounts**
(the only principal is a cryptographic identity), and depin is **not
zero-knowledge** (the coordinator holds decryption keys).

Outcome: clients self-serve through a browser, operators stop running money
commands by hand, and worker operators can see why they are (or are not) being
paid.

## Decisions taken (user)

| Question | Answer |
|---|---|
| Which UIs | Customer console + admin back-office + worker-operator dashboard |
| Client auth | Email + password accounts (real account system, Storj-style) |
| Where served | New standalone coord role(s), own Postgres conn, never migrates |
| Stack | Vue 3 + Vuetify + Pinia + vue-router + Vite, exact-pinned deps |
| Admin auth | Reuses the same email+password `accounts` table (not a separate proxy/group scheme) |
| Identity linking (client address ↔ account) | **Admin-only** for now - no console-side key escrow, no self-serve claim-by-signature yet |
| Build order | Admin back-office ships first, customer console follows |

## Verified findings that shape the design

**No frontend, no accounts.** Zero frontend files in the repo (no
`package.json`, `node_modules`, Vite config, `.vue`). No user/account/login/
password/session/API-key concept. Every gRPC endpoint derives its caller from
`identity.PeerIdentityFromContext(ctx)`. `pkg/auth/auth.go:11` defines only
`RoleNil`/`RoleClient`/`RoleWorker` - no admin role.

**An unwired HTTP scaffold already exists - reuse it.**
- `coord/pkg/router/routegroup.go:9` `RouteGroup` (httprouter wrapper).
- `coord/pkg/responses/` - `ErrorResponse` (errors.go:15), `Success`
  (success.go:7), `WriteJSON` (helpers.go:9), driven by
  `vo.GetHttpStatusCodeFromError` (NotFound→404, Validate→400,
  Unauthorized→401, AlreadyExisted→409, else 500).
- `coord/server/http_routes.go:14` and `coord/server/http_middlewares.go:71`
  are dead code (zero callers). `coord/server/server.go:351`
  `AddHTTPFallback` has zero callers; `publicHTTPListener` (server.go:385) is
  never assigned.
- `httprouter.Router.NotFound` is settable and already used at
  `coord/server/http_routes.go:36` - this is the SPA-fallback hook, so no
  catch-all route conflict.

**depin is NOT zero-knowledge - the key divergence from Storj.** `files.name`
is plaintext `TEXT` (`coord/db/migrations/000002_init_client.up.sql:60`);
`segments.derived_key` is stored coordinator-side (`coord/file/segment.go:57`,
`coord/file/endpoint.go:360`); the edge serves decrypted plaintext
(`edgeserver/handler.go:32`). Consequences:
- A browser file browser can list, preview and download content with no
  client-held passphrase.
- Every passphrase / encryption-wizard / access-grant screen in Storj's
  `web/satellite/` is **out of scope** - that subsystem does not apply.
- Escrowing an account's signing key server-side would add no *new* trust
  assumption if it's ever built (the coordinator already holds the segment
  keys) - noted here because it is the reason that option stays open as
  future work rather than being ruled out; the account model below does not
  use it.

**`files` has no name index and no uniqueness.** Only
`idx_files_contract_id`, `_status`, `_created_at`, `_deleted_at`
(migration 000002:76-79). Names are opaque strings - no path/prefix validation
anywhere in `coord/file/`. So the browser gets a flat list first; S3-style `/`
prefix grouping is a later enhancement, not a schema change.

**Almost every read API a UI needs is missing.**

| Need | Today |
|---|---|
| List contracts for an owner | none - `ContractStore` (coord/storage/contract_store.go:10) is Create/GetByID/GetByIDIncludingDeleted/SoftDelete |
| List files in a contract | none - `file.Store` (coord/file/store.go:23) has no ListByContract |
| List workers | none over RPC/HTTP; in-process `contact.Store` (coord/contact/store.go:10) + debug handler `coord/contact/worker_tags_debug.go:15` |
| Read reputation | none - `reputation.DB` (coord/reputation/service.go:22) is `Update` only, though `reputation.Info` (coord/reputation/reputation.go:60) is rich |
| Repair queue depth, ranged-loop stats, period-close journal, paystubs, admin balances | CLI- or metric-only |

Already wrappable: `coord/billing/endpoint.go` `BillingService`,
`coord/accounting/usage.go:17` `UsageService`, `internal/billing/db.go:188`
`billing.DB`, `internal/compensation/withdrawal.go:277` `WithdrawalDB`,
`internal/compensation/db.go:110` `compensation.DB`.

The CLI commands are the de-facto admin-UI spec: `cmd/coord/billing.go`,
`cmd/coord/compensation.go`, `cmd/coord/withdrawals.go`,
`cmd/coord/deposit.go`, `cmd/coord/periodclose.go`, plus
`coord/overlay/db.go:21` `DisqualifyWorker`/`UndisqualifyWorker`.

**Storj patterns worth porting verbatim.**
- `satellite/admin/authorization.go` - admin auth is a **permission bitmask**
  (`Permission`/`Authorization`/`Has`) with roles `RoleAdmin`, `RoleViewer`,
  `RoleCustomerSupport`, `RoleFinanceManager`, mapped from
  auth-proxy-supplied groups/emails via config lists, least-permissive wins,
  plus a `BypassAuth` dev flag. **No admin password system exists in Storj -
  depin's decision to reuse the email+password `accounts` table for admin is
  a deliberate divergence** (see Architecture); only the bitmask/role
  mechanics are ported, not the proxy-delegated identity source.
- `satellite/admin/auditlogger/` - `Event{UserID, Action, AdminEmail,
  ItemType, Reason, Before, After, Timestamp}`; destructive UI actions require
  a typed reason.
- `satellite/console/consoleweb/server.go:80,739` - customer console assets
  served **from disk** (`StaticDir` + `http.FileServer`), with only
  `error_fallback.html` embedded (server.go:1866) and a "run npm run build"
  log line (server.go:926). `satellite/admin/ui/assets.go` embeds committed
  build output instead - two different mechanisms.
- `satellite/console/` package split: `users.go`, `projects.go`, `mfa.go`,
  `resetpasswordtoken.go`, `registrationtoken.go`, `consoleauth/{sessions,
  token,csrf}`.
- Dependency pinning is exact: 0 of 20 deps in `web/satellite/package.json`
  use `^`/`~`. Admin UI pins `vue 3.5.14`, `vuetify 3.8.6`, `pinia 3.0.2`,
  `vue-router 4.5.1`.

**Peer + test templates.** `coord/relay/peer.go` (+ `coord/relay/config.go`,
`cmd/coord/main.go` wiring) is the standalone-peer precedent: own Config, own
listener, no `coord.base` embedding. Registration is
`pkg/lifecycle/group.go:29` `Item{Name, Run, Close}`; canonical example
`coord/peer.go:252` `setupDebug()`. `internal/testplanet/relay.go` is the
template for adding a peer to the e2e harness, and
`internal/testplanet/reconfigure.go:25` shows the per-peer config hook. DB DSN
comes from `coord/config/config.go:15` `AppConfig.PostgresDsn`.

## Pre-existing bugs that must be fixed first

All three verified in the current tree:

1. **Auth replay window is ~114,000 years.** `pkg/auth/auth.go:61`
   `TimestampWithin(ts int64, windowSeconds int)` multiplies by
   `time.Second`; `pkg/auth/auth_http.go:50` passes `3600000000000`. Any
   validly signed request is accepted forever. Fix: change the parameter to a
   `time.Duration` (single caller) and pass a real window from config.
2. **`recoverPanic` itself panics.** `coord/server/http_middlewares.go:59,63`
   does `err.(string)` on the recovered value. Runtime panics recover as
   `error`, not `string`, so the assertion panics inside the deferred recover
   and the connection dies with no response.
3. **`checkRange` error is untagged.** `coord/billing/endpoint.go:247-256`
   returns a bare `grpcerr.NamedError`, which
   `vo.GetHttpStatusCodeFromError` does not recognise, so an inverted from/to
   range would 500 instead of 400. Fix: `vo.Validate.Wrap(...)` - gRPC
   behaviour is unchanged because `vo.GrpcError` already maps `vo.Validate` to
   `codes.InvalidArgument`.

## Build-system traps

- **`.gitignore` is allowlist-style** (`*` then `!*.go`, `!*.md`, ...). A
  `web/` tree needs explicit `!` entries per extension. Critically, because
  `!*/` re-includes every directory, allowlisting `*.ts`/`*.js`/`*.json` would
  make **`node_modules` committable** - an explicit `node_modules/` ignore
  must follow the allowlist entries. This trap has already bitten the project
  (dropped embed/asm files surfacing only on a fresh clone).
- Some Makefile targets build from an explicit file list, which disables Go vcs
  stamping. Per-component targets live in `Makefile.build` (`build-coord:81`,
  `build-coord-image:142`).
- Release is `docker buildx bake` multi-platform; the Node stage must not make
  `go build ./...` require Node.

## Architecture

### Two new peers, one shared account system

Mirroring Storj's split between `satellite/console/consoleweb` and
`satellite/admin` for *serving* (separate ports/binaries so admin JS is never
shipped to a customer browser), but **not** for auth - both peers authenticate
against the same `accounts`/`account_sessions` tables, unlike Storj (whose
admin has no passwords at all). Session cookies are scoped per peer (own
name/path) so a console session cannot be replayed against the admin peer.

- **`coord admin`** (`coord/admin/`) - admin API + admin SPA. Ships first
  (Phase 1). Never publicly exposed.
- **`coord console`** (`coord/console/`) - customer + worker-operator surfaces
  and their SPA. Ships later (Phase 3+).

Both open their own direct Postgres connection via `AppConfig.PostgresDsn` and
**never run migrations** (same rule as `coord api`/`coord audit`/
`coord repair`).

### Account model - what an email+password user actually owns

New tables:

- `accounts` - `id`, `email` (unique), `password_hash` (bcrypt), `role`
  (`customer` | `worker_operator` | `admin_viewer` | `admin_support` |
  `admin_finance` | `admin_super`), `status`, `full_name`, `mfa_secret`,
  `mfa_recovery_codes`, `email_verified_at`, timestamps.
- `account_clients` - `(account_id, client_address)` join. An account may
  control several client identities.
- `account_workers` - `(account_id, worker_id)` join, for the worker-operator
  dashboard.
- `account_sessions` - opaque token hash, `account_id`, `peer` (`admin` |
  `console`), `expires_at`, `user_agent`, `ip`.
- `account_tokens` - single-use email-verification and password-reset tokens.

**Linking is admin-only, deliberately narrow for now.** A signup creates an
`accounts` row with no attached client or worker; an admin (Phase 2) links a
`client_address` or `worker_id` to it via an audit-logged action, after
verifying the requester out-of-band. This sidesteps two harder problems until
there is a real need for them:

- *Console-side key escrow* (console mints + holds an Ethermint/mTLS
  identity on signup) - would be low-risk to add later, since the
  coordinator already holds `segments.derived_key` for every segment and so
  escrowing an account key adds no new trust assumption, but it is not
  built now.
- *Self-serve claim-by-signature* (user proves ownership of an existing
  client address by signing a console-issued nonce with their Ethermint key)
  - straightforward to add later using `pkg/auth.EthermintVerifier`, but not
  built now.

Both are natural follow-ups after Phase 5, once the admin-linked flow is
validated in production; flagged here so the schema (`account_clients` as a
plain join, no embedded key material) doesn't have to change to add them.

### Auth stack

- Passwords: bcrypt, configurable cost. Shared by both peers.
- Sessions: DB-backed (`account_sessions`), opaque random token, cookie
  `HttpOnly; Secure; SameSite=Lax`, sliding expiry, peer-scoped as above.
  Ports `satellite/console/consoleauth/sessions.go`.
- CSRF: double-submit token on all state-changing requests, per
  `consoleauth/csrf`.
- Email verification and password reset via `account_tokens`, ports
  `registrationtoken.go` / `resetpasswordtoken.go`.
- MFA: TOTP + recovery codes, ports `console/mfa.go`. **Mandatory for every
  `admin_*` role** - unlike Storj's proxy-delegated admin auth, real
  passwords on real admin accounts raise the stakes of a leaked credential,
  so MFA is not optional here the way it is for customers.
- Rate limiting and lockout on login, signup, password reset - applies to
  both peers, tighter thresholds on the admin peer.
- Authorization: port the *mechanics* of `satellite/admin/authorization.go`
  (`Permission`/`Authorization` bitmask, `Has(...)`, per-role constants) but
  source the role from `accounts.role` instead of oauth-proxy groups/emails.
  Drop `BypassAuth` (dev config toggling has more blast radius against real
  password auth than against a proxy) - use a seeded dev admin account
  instead.
- Destructive admin actions require a typed reason and write to
  `admin_audit_log`, porting `satellite/admin/auditlogger`
  (`Event{AccountID, Action, AdminEmail, ItemType, Reason, Before, After,
  Timestamp}`).

### HTTP API shape

- `/api/v0/auth/*` - signup, login, logout, session, verify-email,
  reset-password, MFA.
- `/api/v0/console/*` - customer: account, contracts, files, usage, billing.
- `/api/v0/worker/*` - worker operator: node list, reputation, uptime,
  storage/bandwidth, paystubs, withdrawals.
- `/api/v0/admin/*` - on the admin peer only.

Conventions: JSON only; errors through `coord/pkg/responses` so
`vo.*`-tagged errors map to the right status; **keyset pagination**
(`?limit=&cursor=` returning `{items, next_cursor}`) since offset pagination
over `files`/`segments` would not survive production volume. The coord
`/segment-holders/` debug endpoint already does a keyset scan - copy its shape.

### Frontend layout

Two Vite apps, not four:

- `web/console/` - customer console **and** worker-operator dashboard as two
  route trees in one app (shared session, and one person can be both a client
  and a worker operator). Storj needs separate `web/storagenode/` and
  `web/multinode/` apps only because those are served by the node binaries; in
  depin all worker data lives at the coordinator, so a third build would buy
  nothing.
- `coord/admin/ui/` - admin back-office, mirroring Storj's
  `satellite/admin/ui/` location.

**Serving: from disk for both**, via a `StaticDir` config key +
`http.FileServer`, with `Router.NotFound` doing SPA history fallback and a
`//go:embed`ed fallback page telling the operator to run `npm run build`
(Storj's `error_fallback.html` pattern). This is a deliberate small divergence
from Storj, which embeds committed build output for the admin UI: embedding
would force `dist/` through the allowlist `.gitignore` and re-open the
`node_modules` hazard, and serving from disk guarantees `go build ./...` never
needs Node. Docker bake gains a Node stage that produces `dist/` and copies it
into the image.

Cache headers: hashed assets immutable/long-lived, `index.html` no-cache.
Plus CSP, `X-Content-Type-Options`, `X-Frame-Options`.

API types: hand-written TypeScript DTOs. The repo is gogo-protobuf and the
console DTOs are HTTP-shaped (paginated envelopes, joined account data), not
1:1 with the protos, so proto→TS generation would add toolchain weight for a
poor fit.

## Phases

Each phase is independently shippable. Admin ships first per the build-order
decision above.

**Phase 0 - foundations.** Fix the three pre-existing bugs above (each with a
regression test). Add the missing indexes: `files(contract_id, created_at)`,
`files(contract_id, name)`, `contracts(owner) WHERE deleted_at IS NULL`,
unique `clients(address)` if absent.

**Phase 1 - walking skeleton: admin login.** This is the slice that proves the
whole pipeline end to end, admin-first: `coord/admin/` peer (config,
`NewPeer`, `lifecycle.Item`, `cmd/coord/admin_cmd.go`), `accounts` +
`account_sessions` + `account_tokens` migrations (with `role`), a seed path
for the first `admin_super` account (CLI command or one-shot migration, not a
UI - nothing can log in yet otherwise), MFA-enforced login/logout/session
endpoints, permission-bitmask middleware, `admin_audit_log` table +
`auditlogger` port, `coord/admin/ui/` Vite app with one real authenticated
page (e.g. worker list, since `contact.Store` already backs it). Served from
disk with SPA fallback. Testplanet e2e: seed admin → login → hit one
permission-gated endpoint → audit log entry recorded → logout; wrong-role and
tampered-session cases → 401/403.

**Phase 2 - admin operational surface.** Wrap what the CLI does today as
audit-logged, reason-required endpoints: `coord/overlay/db.go:21`
`DisqualifyWorker`/`UndisqualifyWorker`; billing (`cmd/coord/billing.go`'s
generate-invoices/record-period/credit/balances via
`internal/billing.DB`/`coord/billing`); compensation
(`cmd/coord/compensation.go` via `internal/compensation.DB`); withdrawals
(`cmd/coord/withdrawals.go` via `WithdrawalDB`); deposits
(`cmd/coord/deposit.go`); period-close status
(`cmd/coord/periodclose.go`/`periodclose.StateDB`). New read-only methods for
reputation (`coord/reputation/reputation.go:60` `Info`), repair-queue depth
(`coord/repair/queue/queue.go:54` `Count`), ranged-loop stats
(`coord/rangedloop/stats.go:14`). This is also where the account-linking
endpoint lands: an admin action that writes `account_clients`/
`account_workers` given an account id + address, audit-logged.

**Phase 3 - customer console: read surface.** `coord/console/` peer
(mirrors the Phase 1 admin peer shape), customer-side signup/login/session
(role `customer`, no MFA requirement, still no self-serve identity linking -
customers see nothing until an admin links their address via Phase 2's
endpoint). New repo methods (`ContractStore.ListByOwner`,
`file.Store.ListByContract`) and endpoints for contracts, files, usage,
balance, invoices, deposits - the latter four wrapping the existing
`BillingService`/`UsageService`/`billing.DB` rather than new queries. Refactor
`coord/billing` and `coord/client` endpoint bodies into owner-ID-parameterized
`...For` siblings so gRPC and HTTP share one implementation
(`identity.PeerIdentityFromContext` is gRPC-transport-coupled and cannot
resolve from an HTTP request). `web/console/` Vite app.

**Phase 4 - file browser + worker-operator dashboard.** Browser
upload/download through the existing download-ticket + edge path (viable
without client-held keys per the plaintext-name / escrowed-`derived_key`
finding above). `account_workers` linking (admin-driven, Phase 2's mechanism)
feeds a worker-operator route tree in the same `web/console/` app: uptime/
scores (`coord/contact/worker.go:104` `ComputeScores`), storage/bandwidth
(`accounting.Store.ListWorkerTallies`/`ListWorkerBandwidth`), paystubs
(`compensation.DB.QueryPaystubs`), withdrawal requests
(`WithdrawalDB.ListWithdrawals`/`RequestWithdrawal`).

**Phase 5 - hardening and CI.** Email verification, rate limits, CSP,
pre-compressed assets, `Makefile.build` targets (`build-console-ui`,
`build-admin-ui`) wired into `build-coord-image`, docker bake Node stage,
`.gitlab-ci.yml` lint/type-check/test jobs, `.gitignore` allowlist entries
plus the `node_modules/` guard, pinned Node version.

**Future work (not in this plan's scope, schema left open for it):**
console-side identity escrow on customer signup, self-serve claim-by-signature
for existing out-of-band clients.

Storj admin features with **no depin equivalent** (explicitly out of scope):
project members/invitations, API keys/access grants, coupons/tax IDs/credit
cards, pricing plans, object lock and versioning, bucket event notifications,
custom domains, ObjectMount, compute instances, white-label config, OIDC.

## Critical files

- New: `coord/console/{config,peer,router,auth,handlers_*}.go`,
  `coord/admin/{config,peer,authorization,auditlogger}.go`,
  `cmd/coord/console_cmd.go`, `cmd/coord/admin_cmd.go`,
  `coord/db/migrations/0000NN_*.up.sql`, `web/console/`, `coord/admin/ui/`,
  `internal/testplanet/console.go`.
- Modified: `cmd/coord/main.go` (subcommand wiring),
  `coord/storage/contract_store.go` + `coord/db/contract_repo.go`,
  `coord/file/store.go` + `coord/db/file_repo.go`,
  `coord/billing/endpoint.go` (`...For` methods, `checkRange` fix),
  `coord/client/{endpoint,store}.go` + `coord/db/client_repo.go`,
  `coord/reputation/service.go` (read method), `pkg/auth/{auth,auth_http}.go`
  (window fix), `.gitignore`, `Makefile.build`, bake HCL, `.gitlab-ci.yml`.
- Reused as-is: `coord/pkg/router/routegroup.go`, `coord/pkg/responses/*`,
  `pkg/lifecycle/group.go`, `pkg/debug`, `pkg/telemetry`, `pkg/logship`.
- Templates: `coord/relay/{peer,config}.go`, `internal/testplanet/relay.go`,
  `coord/peer.go:252`, and in Storj `satellite/console/consoleweb/server.go`,
  `satellite/admin/{authorization.go,auditlogger/}`,
  `satellite/console/consoleauth/`.

## Verification

- `go build ./...` **with Node uninstalled / `web/console/dist` absent** - the
  hard constraint; the server must start and serve the embedded fallback page.
- `go test ./pkg/auth/...` - replay-window regression (a 10-minute-stale
  signed request must be rejected).
- `go test ./coord/server/...` - `recoverPanic` regression using a handler
  that panics with an `error`, not a `string`.
- `go test ./coord/billing/...` - inverted range returns 400 over HTTP and
  `InvalidArgument` over gRPC.
- `go test ./internal/testplanet/ -run TestAdmin` /
  `-run TestConsole` - live-listener e2e per phase: seed→login→session→logout
  on each peer; tampered/expired session → 401; wrong-role/wrong-permission →
  403; cross-account access → 403 or 404; audit-log row written for every
  destructive admin action; keyset pagination walks a >1-page fixture without
  repeats or gaps.
- `npm run lint && npm run type-check && npm run test` in `web/console/` and
  `coord/admin/ui/`.
- Manual: `make setup-coord`, seed a contract with files via testplanet or the
  SDK, run `coord console`, sign up, and confirm the browser lists the same
  contracts/files/usage/balance the gRPC path returns for that account.
- `git status` on a **fresh clone** after adding `web/` - confirms the
  allowlist `.gitignore` entries are right and that `node_modules` is not
  tracked.
