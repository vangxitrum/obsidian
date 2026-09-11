---
type: fact
tags: [go-sdk, coverage, testing, storj-port, bugs]
created: 2026-08-14
agent: main
---

# go-sdk coverage 40.6% -> 70.06%, and the bugs found on the way

go-sdk had **no coverage tooling and no CI at all**. Added
`scripts/coverage.sh` (+ `make coverage` / `coverage-check`,
`.coverage-baseline` = 70, `.gitignore` for `cover.out`), modelled on depin's
but with three deliberate deltas: no Postgres guard (go-sdk needs zero infra),
it does NOT abort on a red suite (it prints the number then exits non-zero with
a "this is a FLOOR not a measurement" warning), and a tripwire comparing
packages-that-emitted-blocks against `go list`.

Exclusions: `pkg/pb/**` and `cmd/**` only. **`pkg/infectious` is deliberately
NOT excluded** - it looks like 11k lines but 9.7k of that is one statement-free
generated table; the whole package is 494 statements, and its upstream
(`storj.io/infectious@v0.0.2`, in the module cache) ships tests that port with
a single import rewrite.

## Three P0 blockers had to be fixed before any number was trustworthy

1. `internal/segmentdownload` **panicked ~50% of runs** ("Log in goroutine after
   test completed"). Root cause was production, not test: `piecedownload.Manager.Fetch`
   returns as soon as `requiredCount` pieces land and **leaked every straggler
   goroutine**, each still holding a worker stream. Fixed with `wg.Wait()`
   registered before `cancel()` (defers are LIFO, so cancel runs first and the
   drain is bounded). A panicking binary emits NO coverage blocks, so this was
   silently deleting 129 statements at random.
2. `internal/vo` `TestPieceIDScanNullAndEmpty` failed deterministically:
   `PieceID.Scan` assigned `PieceIDNil` (32 ZERO bytes, a real storable value)
   for a NULL column, so `IsNil()` was false for every inline segment. Now
   assigns nil.
3. `combineErrs` error-chain flattening - see [[download-retry-resume]].

## The big lever: port upstream Storj tests

`storj.io/common` is checked out at `/home/tuan/work/depin-workspace/common`.
Ported with import rewrites only: `pkg/common/sync2` (14 files, 83->448 stmts),
`pkg/ranger` (zero drift), `pkg/common/pkcrypto`, `pkg/common/time2`,
`pkg/common/errs2` (2 of 3 files), `pkg/encryption` (5 files), `pkg/common`
(redundancy/serialnumber), `internal/sync2`, `pkg/infectious` (849 LOC).
Added `testrand.Key`/`Nonce` to match upstream so ported tests need no edits.
Not portable: anything touching `EncryptionParameters.BlockSize` (this SDK has
no per-object block size), `EncSecretBox` (declared but never implemented),
`sync2.Go`/`Concurrently` (absent), `errs2/ignore_test` (needs drpc rpcstatus),
`pkg/eestream` (diverged far past porting).

## Further bugs found by writing tests

- `pkg/identity` `PeerIdentityFromChain` **panicked** (index out of range) on a
  chain shorter than 2. It is fed straight from
  `tls.ConnectionState.PeerCertificates`, i.e. from a remote peer. Added a
  length guard.
- `tlsopts` `removeNils` compacted the CALLER's backing array in place, and
  `Add` passes the same slice to `ClientAdd` then `ServerAdd` - so
  `Add(nil, f, nil)` installed **1 client function and 2 server functions**.
  Now allocates.
- `vo.UUID.Version()` reads byte 15 (random payload), not the version nibble in
  byte 6. Returns a different number for almost every id. Pinned as a known
  defect (`TestUUID_VersionIsBroken`), not fixed - nothing calls it.
- `common.Nonce` Spanner codecs are asymmetric: `EncodeSpanner` emits `[]byte`,
  `DecodeSpanner` only accepts a base64 string and rejects `[]byte`. Survives in
  production only because the Spanner client returns strings.
- `EncryptedPrivateKey.Value()` returns the named type, `Scan` type-asserts on
  `[]byte` - they only round-trip via a sql driver's normalisation.
- `hack/sync-from-depin.sh` would `rm -rf` the repo root's `internal/` and
  `pkg/` and repopulate from `depin/uplink-sdk/`, which no longer exists. Its
  existence check was the only thing preventing total loss. Hard-disabled with
  `exit 1` (kept for reference, not deleted).
- `internal/metrics` `TestConcurrentReaderReportsActiveInterval` is
  timing-flaky **under coverage instrumentation only** (passes 5/5 normally).
  Pre-existing.

## Lint + CI ported from depin (2026-08-14)

go-sdk now has depin's `.golangci.yml` verbatim, `make lint`, and its own
`.gitlab-ci.yml`. Three intentional config differences: no `.dbx.go` path
exclusion, no coord/file revive exclusion, no gosec G204 exclusion.

**go-sdk's CI is much smaller than depin's on purpose.** depin needs a
self-hosted runner, a job token and a global git `insteadOf` rewrite ONLY
because its go.mod replaces cosmos-sdk/ethermint/ibc-go/Gravity-Bridge/go-sdk
onto gitlab.internal. go-sdk has **zero** private deps, so none of that belongs
there - no services, no Postgres, no credentials. Jobs: lint, deps
(`check-no-private-deps`, which nothing had ever run), tidy (diffed, not
mutated), test `-race`, coverage (allow_failure), compile-check across
linux/arm64 + windows/amd64 + freebsd/amd64 + darwin/arm64.

The first lint run produced 68 findings; all fixed, none suppressed. Two were
real rather than cosmetic:
- `connector.DefaultTCPDialer` took a `ctx` and called `net.Dial`, **ignoring
  it** - a cancelled or timed-out download could not abort a TCP connect in
  progress. Now `(&net.Dialer{}).DialContext`.
- `errs2.IsCanceled` compared `err == context.Canceled` instead of
  `errors.Is`, so a wrapped cancellation was not recognised.

Renames (all internal packages, no external blast radius):
`connector.ConnectorConn` -> `connector.Conn`, `dial.DialOptions` ->
`dial.Options`, plus receiver-name consistency in `P2PConnector` and
`CipherSuite`.

Related: [[download-retry-resume]], [[testplanet-coverage-plan]].
