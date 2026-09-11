# Standalone client HTTP API (balance, deposit address, deposit/invoice history, client info)

## Context

Follow-up to "do we support delete files, delete contract" (delete-file: yes, already
implemented; delete-contract: not implemented as of this conversation's initial check, but
already has its own approved design - [[2026-08-04-delete-contract|DeleteContract]]
(soft-delete via `deleted_at`+`status=DELETED`, hard-delete files/segments batched, reject
non-empty unless `force=true`), with implementation apparently underway in parallel this
session (`coord/storage/contract.go` picked up a `Status: pb.ContractStatus_ACTIVE.String()`
field mid-conversation, and [[2026-08-04-testplanet-test-cases]] already documents a Phase 2b
"Deletion (DEL)" section covering DeleteFile/DeleteContract end-to-end tests). Tracked as a
separate plan from this one - do not conflate the two.

User then asked whether depin should support an HTTP server so a client can fetch its own
balance, deposit address, deposit history, "and everything else" instead of calling gRPC.

Exploration found the business logic already exists and is fully implemented, but is
gRPC-only with zero consumers:
- `coord/billing/endpoint.go`: `BillingService` (`GetDepositAddress`, `GetBalance`,
  `ListInvoices`, `ListDeposits`), a thin read layer over `internal/billing.DB`
  (`client_balances`, `client_ledger`, `client_deposits`, `client_invoices` tables).
- `coord/client/endpoint.go`: `ClientService.GetClient`.
- Every RPC is scoped by mTLS peer identity (`identity.PeerIdentityFromContext(ctx).ID`
  *is* the account id) - there is no account parameter anywhere, by design.
- `coord/server/` already has an unwired, dead HTTP-API skeleton clearly built for exactly
  this: `coord/pkg/router` (httprouter wrapper), `coord/pkg/responses`
  (`ErrorResponse`/`Success`, backed by `vo.GetHttpStatusCodeFromError`), and
  `coord/server/http_middlewares.go`'s `authenticationRequest` - a wallet-signature
  (`X-AIOZ-*` headers, `pkg/auth.EthermintVerifier`) HTTP auth middleware, currently
  hardcoded to `RoleWorker` only and never called from anywhere.
- `pkg/auth.RoleClient` is defined but referenced nowhere - scaffolding clearly meant for
  this exact use case.
- Storj's analogous feature (`satellite/console-api.go`, `satellite/console/consoleweb/`)
  confirms the right shape: a **standalone peer/process with its own direct DB connection**
  (not proxied through the main peer's gRPC), thin HTTP handlers wrapping the existing
  account service - which the user independently landed on when asked.

User's decisions (via clarifying questions), locked in for this plan:
1. **New standalone process** (own peer/role), not bolted onto `coord run`/`coord api`.
2. **Purpose**: a convenience read surface for the same clients who could otherwise call
   gRPC directly - not a public/anonymous API.
3. **Auth**: client signs each HTTP request with their identity-dir keypair (wallet-signature
   header scheme). A **companion CLI helper binary** takes `--identity-dir` and does the
   signing so nobody hand-crafts signatures.
4. **Scope**: billing info (deposit address, balance, invoices, deposits) **and** client/account
   info (`ClientService.GetClient`).

## Two pre-existing bugs this work depends on fixing

Both verified by reading the code directly, not assumptions:

**(A) `EthermintVerifier`'s timestamp check is a no-op.** `pkg/auth/auth_http.go:50` calls
`TimestampWithin(tsInt, 3600000000000)`. `TimestampWithin` (`pkg/auth/auth.go:61`) multiplies
its second arg by `time.Second`, so the effective window is ~114,000 years - any validly-signed
request is accepted regardless of timestamp, forever, with zero replay protection. This has
presumably been harmless because the only real caller today (`versioncontrol/api`, ops-only,
`RoleWorker`) isn't account-data-sensitive. It stops being harmless once this scheme
authenticates real client billing/account queries. **Fix as part of this work**: widen
`TimestampWithin(ts int64, window time.Duration)` to take a `time.Duration` directly (its only
caller is `EthermintVerifier.Verify`, so this is a contained change) and pass a real window
(recommend 5 minutes) from `clientapi.Config`. Add a regression test in
`pkg/auth/auth_http_test.go` proving a 10-minutes-stale signed request is now rejected.

**(B) `checkRange`'s error isn't tagged for HTTP mapping.** `coord/billing/endpoint.go:191`
returns a bare `grpcerr.NamedError(...)`, not `vo.Validate.Wrap(...)`.
`vo.GetHttpStatusCodeFromError` only recognizes `vo.*`-tagged errors, so an inverted
`from`/`to` range would silently fall through to HTTP 500 instead of 400. Fix: change
`checkRange` to return `vo.Validate.Wrap(errors.New("`to` must be after `from`"))`
- `vo.GrpcError` already has a case for `vo.Validate` (maps to `codes.InvalidArgument`), so the
existing gRPC behavior is unaffected.

## Architecture

### 1. New package `coord/clientapi/`

Mirrors `coord/relay/`'s process shape (own `Config`, `NewPeer`, `lifecycle.Group`, own
`cmd/coord/<x>_cmd.go` subcommand) but, unlike relay, opens its own direct Postgres
connection (reusing `coord/config.AppConfig`'s `PostgresDsn`) to read the same `clients`,
`client_balances`, `client_ledger`, `client_deposits`, `client_invoices`, `wallets` tables
`coord api`/`coord core` use - same pattern as Storj's standalone `ConsoleAPI` peer. This
process **never migrates** (same rule as `coord api`/`coord audit`/`coord repair`).

```
coord/clientapi/
  config.go            # Address, AuthWindow, AppConfig (DB), Debug, Metrics, LogShip, Identity
  peer.go              # Peer struct + NewPeer(log, ident, config, db) - no coord.base embedding
  auth_middleware.go   # role check + address -> ownerID bridge
  billing_handlers.go  # GetDepositAddress, GetBalance, ListInvoices, ListDeposits
  client_handlers.go   # GetClient
  router.go            # wires coord/pkg/router.RouteGroup + middleware chain
cmd/coord/clientapi_cmd.go   # cobra subcommand, mirrors relay_cmd.go
```

Do not use gin (only precedent is the unrelated `versioncontrol/api`) or hand-roll a bare
mux (only precedent is `relay/status.go`, which has no auth/error-mapping needs). Reuse what
already exists and is fit for purpose: `coord/pkg/router.RouteGroup` for routing,
`coord/pkg/responses` for JSON responses/errors (`vo.GetHttpStatusCodeFromError` already
maps `vo.NotFound`->404, `vo.Validate`->400, `vo.Unauthorized`->401, `vo.AlreadyExisted`->409,
default->500 - no new error-mapping code needed beyond fix (B) above).

### 2. Refactor `coord/billing` and `coord/client` to expose owner-ID-parameterized methods

`identity.PeerIdentityFromContext` is gRPC-transport-coupled (reads `grpcpeer.FromContext`) -
it cannot resolve anything from an HTTP request's context. Rather than duplicating DB-query
logic in HTTP handlers, extract each `Endpoint` method's body (everything after peer-identity
resolution) into an exported, owner-ID-parameterized sibling, called by both the gRPC method
and the new HTTP handler:

- `coord/billing/endpoint.go`: add `GetDepositAddressFor`, `GetBalanceFor`, `ListInvoicesFor`,
  `ListDepositsFor` (ctx, ownerID, ...) -> same response types. Existing `GetDepositAddress`
  etc. shrink to "resolve peerIden, call the `...For` method."
- `coord/client/endpoint.go`: add `GetClientFor(ctx, ownerID)`.

Since these return the same protoc-gen-go structs (which already carry `json:` tags),
`responses.WriteJSON` needs no DTO translation layer.

### 3. Address -> owner-ID bridge

New `client.Store` method, mirroring the existing `GetByID` exactly (`coord/client/store.go`,
`coord/db/client_repo.go`):

```go
GetByAddress(context.Context, vo.AccAddress) (*Client, error)
```

`Client.Address` is already populated at registration from the client's own public key
(`coord/client/types.go` `NewClient`) - the same key family `EthermintVerifier` recovers an
address from. Check `coord/db/migrations/` for the `clients` table and add a unique index on
`address` if none exists (this becomes a hot lookup on every authenticated request).

`coord/clientapi/auth_middleware.go` chains: `recoverPanic -> loggingRequest ->
httpauth.RequireRole(verifier, RoleClient) -> resolveOwner(clientStore) -> handler`, where
`resolveOwner` reads the verified address off context, calls `GetByAddress`, and attaches the
resulting `Client.Id` to context (the HTTP analogue of what gRPC gets for free from
`peerIden.ID`). An address with no registered client -> `vo.Unauthorized` (401, not 404 -
don't leak whether an address exists).

### 4. Auth middleware: new `coord/pkg/httpauth` package

Add `coord/pkg/httpauth/middleware.go`: an exported, **role-parameterized** version of
`coord/server/http_middlewares.go`'s `authenticationRequest` -
`RequireRole(v auth.Verifier, roles ...auth.Role) func(http.HandlerFunc) http.HandlerFunc` -
so `coord/clientapi` isn't stuck either importing an unexported helper from an unrelated peer
or hand-duplicating the header-parsing logic. Leave `coord/server/http_middlewares.go` (dead
code, out of scope) untouched. This creates a third near-duplicate of this header-parsing
logic (alongside `versioncontrol/api/middlewares.go` and `coord/server/http_middlewares.go`) -
acceptable for now, worth a follow-up consolidation later, not blocking.

Payload construction must match the CLI helper exactly: `body + role + address + ts +
publicKeyJSON`, body read via a size-limited reader, `r.Body` reset via `io.NopCloser`
afterward - same as the existing `authenticationRequest`.

### 5. CLI helper binary: `cmd/clientapi-cli/`

A separate small binary (not a subcommand of `cmd/uplink`, to avoid conflating "storage
client" with "billing query tool" - matches the precedent of `cmd/worker-updater/` being its
own binary despite sharing `identity.Config`).

```
cmd/clientapi-cli/
  main.go       # cobra root + subcommands
  signer.go     # loads $IDENTITYDIR/priv_key.json, builds X-AIOZ-* headers, signs
  commands.go   # get-balance, get-deposit-address, list-invoices, list-deposits, get-client
```

Flags: `--identity-dir` (via existing `internal/cfgstruct.IdentityDir`, identical to every
other binary), `--server-address`, `--from`/`--to` (RFC3339) for the two list commands.
Signing logic mirrors `cmd/worker-updater/cmd.go`'s identity-loading pattern +
`internal/version/checker/client.go`'s `authHeaders()` (load priv key ->
`keytool.GetPrivKeyFromJSON` -> derive pubkey/address -> sign `body+role+address+ts+pubkeyJSON`
-> set the five headers). Output: plain JSON passthrough to stdout (server already indents).

### 6. Config

`coord/clientapi/config.go`: `Address` (default `:7790`), `AuthWindow time.Duration`
(default `5m`, feeds fix (A) above), `AppConfig` (DB DSN, same field `coord.Config` already
uses), `Debug`, `Metrics`, `LogShip`, `Identity`. Wire `cmd/coord/clientapi_cmd.go` into
`cmd/coord/main.go`'s `init()` exactly like `relayCmd`.

### 7. Testing (prefer end-to-end per project convention)

Use the existing in-process harness at `internal/testplanet` (+ `coord/db/coorddbtest`):
- Register a client over gRPC (as `coord/billing`'s existing tests do), pairing any
  `testidentity`-pool mTLS identity with a **freshly-generated Ethermint keypair**
  (`ethsecp256k1.GenerateKey()`, same as `pkg/auth/auth_http_test.go`'s `generateTestKey()`)
  for `Register`'s `PublicKey` field - the two keys are independent by design, this works
  without any testplanet changes.
- Fund it via the same billing-repo setup the existing billing endpoint tests use.
- Start `clientapi.Peer` against the same DB, issue real signed HTTP requests (live listener,
  not `httptest`, so the auth middleware is exercised over the wire) for all five endpoints,
  assert parity with the gRPC responses for the same account.
- Negative cases: tampered signature -> 401; valid signature for an unregistered address ->
  401; stale timestamp (10+ min old) -> 401 (proves fix (A) end-to-end); wrong role -> 401;
  `to <= from` -> 400 (proves fix (B) end-to-end).
- Unit test: `coord/db/client_repo_test.go` for `GetByAddress` (found / `ErrRecordNotFound`).

## Critical files

- `coord/billing/endpoint.go` - add `...For` methods, fix `checkRange` tagging
- `coord/client/endpoint.go`, `coord/client/store.go`, `coord/db/client_repo.go` - add
  `GetClientFor`, `GetByAddress`
- `pkg/auth/auth.go`, `pkg/auth/auth_http.go` - fix `TimestampWithin` window bug
- `coord/pkg/httpauth/middleware.go` (new) - role-parameterized auth middleware
- `coord/pkg/router/routegroup.go`, `coord/pkg/responses/*` - reused as-is
- `coord/clientapi/*` (new package, see layout above)
- `cmd/coord/clientapi_cmd.go` (new), `cmd/coord/main.go` (wire subcommand)
- `cmd/clientapi-cli/*` (new binary)
- Reference/template files: `coord/relay/peer.go`, `coord/relay/config.go`,
  `coord/server/http_middlewares.go`, `internal/version/checker/client.go`,
  `cmd/worker-updater/cmd.go`

## Verification

- `go build ./...` across the new/changed packages.
- `go test ./pkg/auth/...` for the timestamp-window regression test.
- `go test ./coord/... -run TestClientAPI` for the new testplanet end-to-end test (all five
  endpoints + the five negative-auth cases above).
- Manual smoke test: run `coord clientapi`, register a test client via the existing gRPC
  path, use `cmd/clientapi-cli` to fetch balance/deposit-address/invoices/deposits/client-info
  and confirm output matches what the gRPC path returns for the same account.
