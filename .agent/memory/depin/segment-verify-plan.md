---
type: decision
tags: [depin, segment-verify, audit, orders, storj-parity]
created: 2026-08-06
updated: 2026-08-07
agent: main
---

**SHIPPED 2026-08-07** (was: plan). All 12 tasks of
`Projects/depin/plans/2026-08-06-segment-verify.md` implemented on branch
`feat/segment-verifier`, uncommitted. `coord/segmentverify/` (library) +
`coord/db/segmentverify_repo.go` (repo) + `cmd/segment-verify/` (binary) +
`internal/testplanet/segment_verify_test.go` (9 e2e scenarios, all pass) +
5-file build/release wiring (`make version-info` lists it) + README. Unit +
DB + testplanet tests all green; smoke-tested for real against a live remote
coordinator (68.183.189.51, real identity at `./identities/`, real 1818-segment
DB) - see "Implementation findings" below.

**User decisions:** targeted + bulk selectors; "align with Storj" wherever Storj has an opinion;
standalone binary (not a `coord` subcommand, despite `cmd/coord/compensation.go`'s documented
one-shot-command convention); both RS and CLONE. Read-only - no reputation writes, no repair enqueue.

## Non-obvious findings worth keeping

**Storj's `Exists` RPC is dead on both sides.** `storj/storagenode/piecestore/endpoint.go:206`
returns `Unimplemented` ("exists is no longer supported"), and segment-verify's Exists path was
removed in the `dd6bc10a9` revert. Its README still documents `--verify.version-with-exists` - stale.
So current Storj = 1-byte `Download` probe, and depin needs **no worker-side change**. Do not build
an `Exists` RPC on the strength of Storj having one.

**Order limits must be minted PER PROBE, not per segment.**
`worker/piecestore/usedserials/table.go:77` `Add(ctx, coordinatorID, serialNumber, expiration)` keys
replay protection on **(coordinator, serial) only - not piece id**. `NewSigner`
(`coord/order/signer.go:60-92`) mints one fresh keypair + serial per call, so minting one limit for a
whole segment's piece list means every piece shares a serial - and the second piece landing on the
same worker is rejected as a duplicate serial. Silent and data-dependent. The correct pattern already
exists in production: `coord/audit/verifier.go:563` calls
`CreateAuditGetOrderLimits(ctx, root, pieceSize, []order.PieceLocation{loc})` with a ONE-element slice.

**A successful `Fetch` can return zero bytes.** `worker/piecestore/endpoint.go:870-872` clamps the
requested range (`if length <= 0 || offset+length > pieceSize { length = pieceSize - offset }`), so a
zero-length stored piece streams 0 bytes and EOFs cleanly -> `(([]byte{}), nil)`. **`err == nil` is
not a success signal**; must assert `len(data) == 1`. depin's own audit already guards this at
`coord/audit/verifier.go:780-782`. Relatedly, pass `length = 1` and never `0`:
`signSingleUseOrder` (`coord/audit/fetch.go:135-137`) clamps `amount <= 0` UP to the full piece size.

**Two order-minting entry points, only one is right for verification.**
`CreateAuditGetOrderLimits` (`coord/order/service.go:179`) takes pre-resolved `[]PieceLocation` and
mints dense limits. `CreateAuditOrderLimits` (`:302`) resolves via `GetWorkersForAudit` and silently
**drops offline + disqualified holders before minting**, returning a slice indexed by piece number
with nil holes. Anything that needs to probe or report offline holders must use the former plus the
unfiltered worker lookup - `GetWorkersForAudit` (`coord/db/worker_repo.go:114`) filters
`disqualified_at IS NULL AND deleted_at IS NULL AND last_seen >= window`.

**Concurrency cannot follow Storj's numbers.** Storj defaults `--service.concurrency=1000` over plain
TCP. depin dials through `connector.NewHybridConnector(tcp, p2p)` and every NAT'd worker is reached
through one shared relay IP; `internal/grpcutil/connector/p2p_connector.go:79-91` documents libp2p's
per-IP connection cap. 1000 concurrent batches through one relay exhausts it and mass-reports false
offlines. Plan defaults to 64. See [[sdk-perip-connlimit]].

**`pool.WithForceDial` in depin is not Storj's force-dial.**
`internal/grpcutil/pool/pool.go:278-303` takes from the cache first; it only eagerly materializes the
conn at Get time. Do not use it to force a redial.

**Revocation DB must default to `memory://` for any coord-adjacent CLI tool.**
`Extensions.Revocation` defaults true and `RevocationDBURL` to `bolt://$CONFDIR/revocations.db`;
`newBoltStore` (`internal/revocation/store.go:44-49`) takes an **exclusive flock** with a 1s timeout.
A running coordinator holds it, so `--config-dir <coord dir>` dies after 1s - in exactly the scenario
such a tool exists for. `coord/peer.go:142 revocationDB()` memoises for the same reason.

**`.gitignore` is an allowlist and silently eats `.csv`.** Confirmed with `git check-ignore`:
`coord/segmentverify/testdata/segments.csv` is dropped (`*` then `!*.go`, `!*.md`, ...). Build CSV
test fixtures in-code, not as `testdata/*.csv`. Generalises the existing
[[depin-gitignore-allowlist-gotcha]].

**`classifyErr` doc bullet is stale, code is deliberate.** `coord/audit/verifier.go:891-916` - the
bullet list says "anything else -> unknown" but it returns `AuditOffline`, and the trailing comment
("Dial failures surface as plain wrapped errors; treat as offline") shows that is intended. Fix the
comment, not the behaviour.

**CLONE needs one shared helper, not a parallel code path.** The probe is scheme-agnostic;
`file.RedundancyScheme.PieceSize` (`coord/file/file.go:52-56`) already short-circuits CLONE to
`encryptedSize`, and the health threshold of 1 is already established at
`coord/file/endpoint.go:1075-1079`. The one real divergence from Storj: Storj's `CreateBatches`
SKIPS a segment when `len(Pieces) < Check`, which is wrong for depin where narrow (3-copy CLONE)
schemes are normal - clamp `Retry = min(Check, len(Pieces))` instead.

**testplanet can cover this end to end.** `Worker.DropCoordPieces` (`internal/testplanet/helpers.go:126`)
kills pieces on a live worker (hashstore is append-only, so this is the sanctioned way),
`Planet.PieceHolders` (`:135`), `Coord.AuditSegmentForFile` (`:94`), `Planet.StopPeer` (`planet.go:340`),
`testplanet.CloneRedundancy` / `DefaultRedundancy` (`planet.go:65-86`). This is why the library goes
in `coord/segmentverify/` and not `package main` - `internal/` cannot import a main package, which is
how Storj gets away with a single-package tool. `Coord.MarkWorkerGone` is NOT the right helper for a
"disqualified, skip dialing" test - it only backdates `last_seen`/`is_online` (repair's online-window
semantics); disqualify directly via `UPDATE workers SET disqualified_at = ...`.

## Implementation findings (2026-08-07)

**The embedded-struct spike FAILED, as flagged as the highest-uncertainty item.**
`type segmentVerifyScan struct { auditSegmentScan `gorm:"embedded"`; FileID vo.UUID }` compiles and
queries without error, but every field from the embedded struct decodes to its zero value - only
`FileID` (declared directly, not via embedding) comes back populated. Root cause: gorm's schema
parser treats an anonymous `gorm:"embedded"` field as an association rather than flattened columns
when that field's type itself implements `TableName()` (promoted from the embedded type). Verified
against a live Postgres row, not assumed. Fallback used: flat 11 fields, own `TableName()`, reuse
`auditPieces()` - exactly the plan's own documented fallback.

**Pre-existing, unrelated bug: `coord/jobq/config.go:41`'s `MaxAttempts int32` panics `cfgstruct.Bind`
("invalid field type: int32") and kills `make setup-coord` (and every other `coord` subcommand) at
`init()`.** `internal/cfgstruct/cfgstruct.go`'s numeric-type switch has no `int32` case (only
`int`/`int64`/`float64`). Confirmed via `git stash` that this branch never touched `cmd/coord` or
`coord/jobq` - fully pre-existing. Blocks local `make setup-coord`; worked around by using a real
remote coordinator's identity + DB directly instead. Not fixed (out of scope for this plan) - flagged
to the user, who chose to route around it rather than have it fixed inline.

**`cfgstruct.ConfigVar` is the mechanism for per-leaf-command config defaults**, when the same struct
(e.g. `segmentverify.ServiceConfig`) is bound to multiple cobra commands that need different defaults
for the same field. A plain Go field assignment in a cmd's `init()` does NOT work - `process.Bind`'s
struct-tag-driven default always runs after any `init()` and unconditionally resets the field.
Instead: tag the field `default:"${SOME_VAR}"` and pass `cfgstruct.ConfigVar("SOME_VAR", value)` as a
`BindOpt` per call site (`internal/testplanet/coord.go` already does this for `${HOST}`/`${TESTINTERVAL}`).
Used here for `run`'s `--service.check=3` vs `check`'s `--service.check=0`.

**A shared `*NodeVerifier`'s `reportPiece` callback is last-assignment-wins**, since both `NewService`
(wires it to filter into `problem-pieces.csv`, non-success only) and `check`'s own recording wrapper
(wants every outcome, including SUCCESS, for its live table) mutate the same field on the same
instance. Whichever runs second clobbers the other. Found by actually running the binary against real
data (`check segment` showed "probed 0 of N" even though real dials happened) - not something a
mocked unit test would ever catch. Fixed by calling `NewService` first, then the recording
`SetReportPiece` after.

**CSV writers (`CSVWriter`/`pieceCSVWriter`) now open their file lazily on first `Write`, not at
construction.** `NewService` always builds all three writers (`segments-not-found.csv`,
`segments-retry.csv`, `problem-pieces.csv`) regardless of which cobra leaf is running; `check` never
calls `ProcessSegments` (the only thing that ever calls `Write`), so eager `os.Create` littered three
empty files in the CWD on every `check` invocation. `run`'s behavior is unaffected - `ProcessSegments`
always calls `Write` at least once per batch (even with zero problem segments, to emit the header),
which is exactly when the file needs to exist.

**A tiny test redundancy scheme (RS with `SuccessThreshold == TotalShares`) makes the retry-pass
heuristic self-heal by accident.** If every piece already gets tried in pass 1 (Check == TotalShares),
the retry pass's "reverse the piece list, probe a fresh one" logic has no genuinely untried
alternative left - it just re-probes an already-succeeded piece, silently resolving the segment fully
(Retry -> 0) even though the originally-offline worker was never actually recovered. This is faithful
Storj behavior, not a depin bug, but it means an "offline worker lands in retry.csv" test needs at
least two offline workers, pinned to both the first-pass slot (`Pieces[:Check]`) and the post-reversal
retry-pass slot (`Pieces[:Check]` of the reversed list, i.e. originally the *last* piece) - one
offline worker alone will very likely get "fixed" by the retry pass finding a different, alive piece
(which is the desired, separately-tested behavior - see the "retry pass probes a different piece"
scenario).

**Local dev `coord-db` (docker, port 5445) was 5 migrations behind the repo (version 38 vs 43) before
this session** - missing `000042_segments_created_at`, which this feature depends on entirely (every
scan struct maps that column, so literally any query fails with `column "created_at" does not
exist`). Migrated 38→43 with the user's explicit approval (took a `pg_dump` backup first); row counts
(5940 files / 6242 segments / 1200 workers) unchanged after. Real production/demo data lives on a
separate host, `demo` in `~/.ssh/config` (68.183.189.51) - its Postgres is also exposed on 5445 and
was already at migration 43. Confirmed via a real `check segment <id>` run using the real identity at
`./identities/` (CA+leaf bundle matching `identity.Config`'s expected filenames): resolved 60 real
workers, minted 60 real GET_AUDIT order limits, dialed all 60 over real p2p/relay circuits - all
`TIMED_OUT` (this sandbox has no route into AIOZ's relay mesh, expected), verdict `INCONCLUSIVE`,
correct exit code 1. Proves the full pipeline end to end against real infrastructure, independent of
network reachability.

Related: [[worker-hashstore-pieces-endpoint]], [[coord-audit-metrics-instrumentation]],
[[coord-repair-metrics-instrumentation]], [[depin-gitignore-allowlist-gotcha]], [[cfgstruct-bindable-types]].
