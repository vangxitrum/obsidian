---
type: decision
tags: [edgeserver, go-sdk, uplink-sdk, testplanet, gogo-protobuf, migration]
created: 2026-07-10
agent: main
---

Migrated `edgeserver` + `internal/testplanet` off the in-repo `aioz-depin/uplink-sdk` onto the
standalone `aioz-depin/go-sdk` module (GitLab `gitlab.internal/aioz-depin/go-sdk`, host-less
module path pulled in via `require`+`replace` pinned to a `main` pseudo-version, no semver tags
— same pattern as the existing `10.0.0.50/...` forks). Deleted `uplink-sdk/` entirely (zero
remaining importers). Branch `feat/build-edgeserver-with-sdk`, uncommitted per user's no-auto-commit rule.

Wiring recipe (repeatable for future go-sdk bumps): clone go-sdk, compute
`v0.0.0-<UTC-commit-date>-<12-char-short-sha>` pseudo-version, `go mod edit -require=... -replace=...`,
`go mod tidy`. `go mod tidy` prunes the `require` line to a placeholder zero pseudo-version
(`v0.0.0-00010101000000-000000000000`) if nothing currently imports the package — normal, matches
existing `AIOZNetwork/go-aioz` require line; it re-resolves once an import exists.

**Real blocker hit (not anticipated by the plan): gogo-protobuf duplicate registration panic.**
go-sdk vendors its own generated copies of `pkg/pb/{shared,coord/{placement,file,storage,client},worker/piece}`
instead of importing depin's own `pkg/pb/*` — same proto full names, different Go types. Only
`proto.RegisterEnum` panics on duplicate (message `RegisterType` just logs a warning, non-fatal).
This only breaks binaries that link BOTH trees in one process — `internal/testplanet` (spins an
in-process depin coordinator+workers *and* drives them via the go-sdk client) hit it hard at
package-init time, before any test ran. `edgeserver` itself is unaffected (checked its full
dep list — only pulls `go-sdk/pkg/pb/*`, never depin's own). Fixed upstream in go-sdk commit
`bb4f12b3ecdc` ("fix(proto): drop RegisterEnum calls from vendored packages"); after re-pinning
to that pseudo-version, full `internal/testplanet` suite (16 tests, `DEPIN_TEST_POSTGRES` against
the already-running `coord-db` docker container on `localhost:5445`, db `hub`, admin/admin123)
passes clean. One test (`TestSettlementRejectsOverAllocation`) failed once under full-suite
parallel run then passed on retry twice — pre-existing shared-postgres-db race between tests, not
migration-related, don't chase it as an SDK regression.

**How to apply:** if go-sdk is bumped again and something mysteriously panics with
`proto: duplicate enum registered` at test/binary init, check whether go-sdk re-vendored pb
types that collide with depin's own `pkg/pb/*` again — this is a go-sdk-repo-side generation bug,
not fixable from depin's side, needs a go-sdk patch.
