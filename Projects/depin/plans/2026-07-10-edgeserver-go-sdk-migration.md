# Migrate edgeserver (and testplanet) from in-repo `uplink-sdk` to `../go-sdk`

## Context

`edgeserver` currently downloads files through the in-repo package `aioz-depin/uplink-sdk`
(same Go module). We want it to use the standalone module `aioz-depin/go-sdk`, now released to
the internal GitLab at `gitlab.internal/aioz-depin/go-sdk` and imported as a normal versioned
dependency (via a `replace`, since its go.mod path is host-less — see step 1).
`go-sdk` is a cleaned-up port of `uplink-sdk` (Storj-derived): same package name `uplinksdk`,
same public method names, no private `10.0.0.50` deps, shares depin's `gogo/protobuf` replace.

Decision (confirmed with user): **full migration** — also move `internal/testplanet` onto
`go-sdk` and **delete the in-repo `uplink-sdk/` directory**. Scope is limited to the SDK swap
plus the build-tag fix; we are **not** filling the unrelated docker gaps (missing
`Dockerfile.edgeserver` / `dev/edgeserver`).

The only two in-repo Go importers of `uplink-sdk` are `edgeserver` and `internal/testplanet`
(everything else is comment-only). Once both are migrated, `uplink-sdk/` has zero importers.

### The one real friction point
`go-sdk` moved its identifier types under `go-sdk/internal/vo`, so an `aioz-depin` sibling
caller **cannot name** `go-sdk`'s `vo.UUID`. All other types that appear in the public API
(`identity.FullIdentity`, `common.EncryptionParameters`, all `pb` types) stayed under
`go-sdk/pkg/*` and remain nameable. testplanet never calls `client.Identity()`/`PieceKey()`
(it makes its own on disk), so the **only** break is `vo.UUID`, at exactly three spots plus a
new 3-value `UploadFile` return. Bridges (no naming of go-sdk's internal type required):
- out: `vo.UUID(id)` — value conversion, both are `type UUID []byte`.
- in: `uplinksdk.ParseUUID(fileID.String())` — go-sdk's public string constructor.
- round-trip is safe: depin & go-sdk `vo.UUID.String()` are byte-identical (`uuid.UUID(id.Bytes()).String()`).

## Changes

### 1. `go.mod` (module wiring)
`go-sdk`'s released go.mod still declares `module aioz-depin/go-sdk` (a host-less path), and the
GitLab repo has **no semver tags — only `main`**. So, exactly like depin's existing
`10.0.0.50/...` forks, it is pulled in via a `require` on the declared path plus a `replace`
mapping that path to the GitLab repo at a pinned `main` pseudo-version. Import lines stay
`aioz-depin/go-sdk` (no rename).

Prereqs already satisfied on this machine: `GOPRIVATE=gitlab.internal/*,10.0.0.50/*` and git
`insteadOf` rewriting `https://gitlab.internal/` → `ssh://git@gitlab.internal/` (fetch over ssh,
no sumdb). Wiring recipe:

```bash
# resolve main's pseudo-version from the released repo
TMP=$(mktemp -d); git clone --depth 1 ssh://git@gitlab.internal/aioz-depin/go-sdk.git "$TMP"
PSEUDO="v0.0.0-$(TZ=UTC git -C "$TMP" show -s --date=format-local:%Y%m%d%H%M%S --format=%cd HEAD)-$(git -C "$TMP" rev-parse --short=12 HEAD)"
# wire depin go.mod (adds to require + replace blocks)
go mod edit -require="aioz-depin/go-sdk@${PSEUDO}"
go mod edit -replace="aioz-depin/go-sdk=gitlab.internal/aioz-depin/go-sdk@${PSEUDO}"
go mod tidy
```

The `replace` target's go.mod declares `module aioz-depin/go-sdk`, matching the left side, so Go
accepts it. Note: a bare `go get gitlab.internal/aioz-depin/go-sdk@main` will **fail** on the
module-path mismatch — the require+replace pair with an explicit pseudo-version is required.

`go-sdk` pins some deps newer than depin (go-libp2p v0.46.0, quic-go v0.57.1, grpc v1.74.2,
several `golang.org/x/*`). MVS will bump depin to the max. **This is the primary risk** — the
bumps must not break depin's cosmos/ethermint stack; caught by the full build in verification.

### 2. `edgeserver/server.go` — one import line
- `server.go:10`: `uplinksdk "aioz-depin/uplink-sdk"` → `uplinksdk "aioz-depin/go-sdk"`.
- Everything else is unchanged: `New`, `WithIdentityDir`, `WithCoordPeerURL`, `WithLogger`,
  the `*uplinksdk.Client` field, and `handler.go`'s `DownloadByTicket` call all have identical
  signatures in go-sdk. `handler.go` imports nothing from the SDK (uses the struct field) → untouched.
- Optional: refresh the "uplink-sdk" wording in the doc comments (`server.go:26`, `config.go:5`).

### 3. `internal/testplanet/uplink.go` — import + `vo.UUID` bridges
- `uplink.go:18`: `uplinksdk "aioz-depin/uplink-sdk"` → `uplinksdk "aioz-depin/go-sdk"`.
- Keep the existing `aioz-depin/internal/vo` and `aioz-depin/pkg/identity` imports (still used for
  the `Upload`/`Download` signatures and for writing the on-disk identity/priv/piece-key files).
- `Upload` (lines 55, 67): `UploadFile` now returns 3 values, and its `vo.UUID` is go-sdk-internal:
  - `id, err := client.Client.UploadFile(...)` → `id, _, err := client.Client.UploadFile(...)`
  - `return id, nil` → `return vo.UUID(id), nil`
  - `contractID` (line 47, `:=`) stays inferred as go-sdk's `vo.UUID` and feeds
    `UploadParams.ContractID` directly — no change needed; `testPlacementID` is already `int32`.
- `Download` (lines 74-76): convert the depin `vo.UUID` param into go-sdk's via the public parser:
  ```go
  sdkID, err := uplinksdk.ParseUUID(fileID.String())
  if err != nil {
      return nil, errs.Wrap(err)
  }
  var buf bytes.Buffer
  if err := client.Client.DownloadFile(ctx, sdkID, &buf); err != nil {
      return nil, errs.Wrap(err)
  }
  ```
- The `Upload`/`Download` helper signatures keep depin's `vo.UUID`, so **none of the ~8 consuming
  test files change** (upload_test, download_test, clone_test, audit_test, accounting_invalid_test,
  storage_tally_test, bandwidth_rollup_test, worker_stored_bytes_test).

### 4. `Makefile.build` — add `purego` tag to edgeserver targets
go-sdk's `pkg/infectious` needs the `purego` build tag on amd64 (`addmul_amd64.go: //go:build !purego`).
The current edgeserver targets pass no tags. Match the coord/worker pattern:
- `build-edgeserver` (line 204):
  `@CGO_ENABLED=1 GOOS=$(GOOS) GOARCH=$(GOARCH) go build -tags '$(GO_BUILD_TAGS)' -ldflags=$(GO_LDFLAGS) -o $(BIN_DIR)/edgeserver ./cmd/edgeserver/*.go`
- `setup-edgeserver` (line 207): `@go run -tags '$(GO_BUILD_TAGS)' ./cmd/edgeserver/*.go setup --log.encoding pretty`
- `run-edgeserver` (line 210): `@go run -tags '$(GO_BUILD_TAGS)' ./cmd/edgeserver/*.go run --config-dir ./dev/edgeserver`

### 5. Delete `uplink-sdk/` + clean stale references
- `rm -rf uplink-sdk/` (no Go importers remain after steps 2-3).
- Non-blocking hygiene (already broken today, but tidy up while here):
  - `gen_identities_dev.sh:75` and `gen_identities_prod.sh:94` reference `uplink-sdk/cmd/gen-piece-key`,
    which **does not exist** — leave the scripts or repoint to `cmd/keytool new-piece` (used elsewhere).
  - `.air.toml` excludes of `uplink-sdk/test` in `dev/coord/*.air*.toml` and `dev/worker{1,2}/.air.toml`
    become dangling — harmless; drop them if convenient.

## Verification

No sibling checkout or symlink needed — `go-sdk` is fetched from GitLab via the require+replace
in step 1 (GOPRIVATE + ssh `insteadOf` already configured). Run from the depin repo root:

1. **Module resolves & whole tree builds** (this catches the MVS-bump risk):
   `go mod tidy && go build -tags purego ./...`
2. **edgeserver builds via Make** (proves the tag fix): `make build-edgeserver`
3. **End-to-end guard — testplanet upload→download through go-sdk** (the real behavior test; spins an
   in-process coordinator + workers and round-trips a file):
   `go test -tags purego ./internal/testplanet/ -run 'Upload|Download' -count=1`
   Also run the broader testplanet suite that exercises the `Uplink` helper
   (`clone_test`, `audit_test`, accounting/tally tests) to confirm the `vo.UUID` bridges behave.
4. **edgeserver smoke (optional, heavy)**: `make setup-edgeserver` then `make run-edgeserver` against a
   running dev coordinator, and `GET /download?ticket=…` a ticket minted by an uplink — confirms
   `DownloadByTicket` works over go-sdk. Skipped if a full dev cluster isn't up; step 3 already covers
   the go-sdk download path end-to-end.

## Out of scope (noted, not done)
- Migrating other components — none import `uplink-sdk` in code (comment-only mentions).
- Adding `Dockerfile.edgeserver` / `dev/edgeserver` (pre-existing gaps, per user decision).
