# Plan: segment-verify (manual segment health checking)

## Context

Today there is no way for an operator to answer "does this segment still exist on the network?"
on demand. The pieces that exist are all *automatic and sampled*: `coord/audit` picks random
segments via a reservoir and verifies a random stripe; `coord/repair/checker` classifies segments
from **DB state only** (`ClassifySegmentPieces` never contacts a worker). Neither can be pointed at
a specific file, and neither answers the operator question directly.

This adds a standalone `segment-verify` tool, ported from Storj's `cmd/tools/segment-verify`, that
probes every holder of a segment and reports which pieces are actually still there. It is
**read-only** — it never writes reputation and never enqueues repair.

Decisions locked with the user:

| | Decision |
|---|---|
| Scope | targeted selectors **and** bulk scan |
| Alignment | follow Storj wherever Storj has an opinion |
| Probe | 1-byte `Download` with a `GET_AUDIT` order limit — **no** new worker RPC |
| Write-back | read-only (Storj passes `orders.NewNoopDB()`) |
| Packaging | standalone `cmd/segment-verify` binary |
| Algorithms | both `REED_SOLOMON` and `CLONE`; inline segments skipped |

On the probe choice: Storj's storagenode *has* an `Exists` RPC in the proto, but the handler at
`storj/storagenode/piecestore/endpoint.go:206` returns `Unimplemented` ("exists is no longer
supported"), and segment-verify's `Exists` path was removed too. Current Storj is 1-byte `Download`,
period. So no worker-side change is needed here — which also sidesteps the fact that fleet
auto-update is currently inert.

---

## Verified facts this plan depends on

Each of these was checked against the tree, not assumed.

1. **Per-probe order minting is already the production pattern.**
   `coord/audit/verifier.go:563` calls
   `CreateAuditGetOrderLimits(ctx, seg.RootPieceID, pieceSize, []order.PieceLocation{loc})` with a
   one-element slice. Reuse this shape.

2. **Minting per segment instead would be a latent bug.** `NewSigner` (`coord/order/signer.go:60-92`)
   makes a fresh `vo.NewPieceKey()` + `CreateSerial()` per call, so one limit per segment means one
   serial shared across its pieces. `usedserials.Table.Add` (`worker/piecestore/usedserials/table.go:77`)
   keys on **`(coordinatorID, serialNumber)` only — not piece id**, so the second piece landing on the
   same worker is rejected as a duplicate serial. Data-dependent and silent. **Mint per probe.**

3. **`err == nil` is not a success signal.** The worker clamps the range
   (`worker/piecestore/endpoint.go:870-872`: `if length <= 0 || offset+length > pieceSize { length = pieceSize - offset }`),
   so a zero-length stored piece streams 0 bytes and EOFs cleanly. Must assert `len(data) == 1`.
   depin's own audit already does this (`coord/audit/verifier.go:780-782`).

4. **Pass `length = 1`, never `0`.** `signSingleUseOrder` (`coord/audit/fetch.go:135-137`) clamps
   `amount <= 0` up to the full `limit.Limit`, i.e. it would settle the whole piece size.

5. **`ClassifyFetchError` falls through to `AuditOffline`** (`coord/audit/verifier.go:891-916`).
   The stale bullet in its doc comment says "unknown", but the trailing comment ("Dial failures
   surface as plain wrapped errors; treat as offline") shows the behaviour is deliberate. Reuse the
   classifier as-is; `UNKNOWN_ERROR` will simply be rare. Fix the stale bullet in a separate commit.

6. **`file.RedundancyScheme.PieceSize`** (`coord/file/file.go:52-56`) short-circuits CLONE to
   `encryptedSize`. Use it, not `coord/audit`'s unexported `rsPieceSize`, and both algorithms are
   handled by one call.

7. **testplanet can cover this end to end**: `Worker.DropCoordPieces` (`internal/testplanet/helpers.go:126`)
   kills pieces on a live worker, `Planet.PieceHolders` (`:135`) maps pieces to holders,
   `Coord.AuditSegmentForFile` (`:94`), `Coord.MarkWorkerGone` (`:313`), `Planet.StopPeer`
   (`planet.go:340`), and `testplanet.CloneRedundancy` / `DefaultRedundancy` (`planet.go:65-86`)
   give RS and CLONE fixtures.

8. **`.gitignore` is an allowlist** (`*` then `!*.go`, `!*.md`, …). `.csv` files are **silently
   ignored** — `git check-ignore` confirms `coord/segmentverify/testdata/segments.csv` is dropped.
   Build CSV fixtures in-code (which is what Storj's `csv_test.go` does anyway); do not add a
   `testdata/*.csv`.

---

## Package layout

Library in `coord/segmentverify/`, thin `package main` in `cmd/segment-verify/`. Storj puts
everything in `package main`; do not copy that — `internal/testplanet` cannot import a `main`
package, and the DB queries must live in `coord/db` to reuse the unexported `auditSegmentScan`
projection. This mirrors how `coord/audit` declares interfaces that `coord/db/audit_repo.go`
implements.

```
coord/segmentverify/
  segmentverify.go   Error, mon, Segment, Status, Batch, WorkerTarget
  config.go          ServiceConfig, VerifierConfig, DateFlag
  service.go         Service, ProcessSegments, RemoveDeleted, worker-list loading
  batch.go           CreateBatches, selectOnlinePieces, sortPriorityToFirst, removePriorityPieces
  process.go         Verify (two passes), VerifyBatches, worker resolution cache
  verify.go          NodeVerifier (the probe), rateLimiter
  source.go          SegmentSource + range / contract / csv / targeted sources
  aliasset.go        WorkerAliasSet, workerAliasExpiringSet
  csv.go             CSVWriter, pieceCSVWriter, SegmentCSVSource
  report.go          verdict computation + table/JSON output
  README.md
coord/db/
  segmentverify_repo.go
cmd/segment-verify/
  main.go run.go check.go summarize.go
```

Not `internal/segmentverify` — `internal/` holds leaves with no coordinator deps
(`internal/billing`, `internal/vo`); this imports `coord/order`, `coord/placement`, `coord/audit`,
`coord/file`.

Config: embed the whole coord config, following `cmd/coord/compensation.go:56` —
`var segmentVerifyCfg = &struct{ coord.Config }{}`. This gives exact flag/viper-key parity with the
`config.yaml` that `coord setup` writes, so `--config-dir <coord dir>` resolves `server.*`,
`identity.*`, `app-config.db.postgres-dsn` and `order.*` with zero drift. Importing package `coord`
does not bind ports — only calling `server.New` does.

---

## Command surface

depin has no metabase and no buckets; the analogue of a Storj bucket is **contract → files →
segments** (`contracts` ← `files.contract_id` ← `segments.file_id`).

```
segment-verify
├── run                                     bulk; CSV output; the Storj shape
│   ├── range     --low <hex> --high <hex>
│   ├── contracts --contracts-csv <path> | --contract-id <uuid>...   ← Storj `buckets`
│   ├── read-csv  --input-file <path>
│   └── ranges    --count N                 prints shard boundaries
├── check                                   targeted; table/JSON; exit code = verdict
│   ├── segment  <segment-uuid>...
│   ├── file     <file-uuid>...
│   └── contract <contract-uuid>
└── summarize-log <file>
```

- `--low/--high`: Storj's hex-prefix semantics (strip dashes, hex-decode, left-align into 16 bytes,
  `high` exclusive), also accepting a full canonical UUID. Segment ids are UUIDv7, so ranges are
  time-ordered and "verify recent uploads" is naturally expressible.
- `run ranges --count N` wraps `rangedloop.CreateUUIDRanges(n)` (`coord/rangedloop/rangesplitter.go:28`)
  so a full-cluster run can be sharded across processes. Two lines, and there is no other correct
  way to get the boundaries.
- `read-csv` parses column 0 as a segment UUID and ignores the rest, so it round-trips this tool's
  own not-found/retry CSVs. Skip a leading row whose column 0 is not a UUID.
- `check` defaults to `--service.check=0` (probe **every** piece) — a targeted check exists to
  produce a verdict, and 3-of-80 cannot. `run` keeps Storj's default of 3.
- `check` exit codes: `0` healthy, `1` at least one segment degraded/lost, `2` tool error.

---

## Segment source layer

Currency type wraps the existing rangedloop segment:

```go
type Segment struct {
    rangedloop.Segment          // coord/rangedloop/segment.go:22
    FileID vo.UUID
    Status Status
}
```

Projection reuses `auditSegmentScan` (`coord/db/audit_repo.go:57-96`) rather than duplicating it —
it carries the `file.RedundancyScheme` / `file.Pieces` `sql.Scanner` wiring that decodes the packed
BIGINT scheme and the RLE piece blob:

```go
// coord/db/segmentverify_repo.go
type segmentVerifyScan struct {
    auditSegmentScan `gorm:"embedded"`
    FileID vo.UUID   `gorm:"column:file_id"`
}
```

New repository methods (`SegmentVerifyRepository`): `ListSegmentsRange`, `ListSegmentsByIDs`,
`ListSegmentsByFile`, `ListSegmentsByContract`, `ExistingSegmentIDs` (one batched query replacing
Storj's N point queries in `RemoveDeleted`), plus `AllWorkerTargets` / `WorkerTargetsByAliases` /
`WorkerTargetsByIDs`.

Reuse vs new:

| Need | Existing | Verdict |
|---|---|---|
| keyset pagination shape | `coord/db/rangedloop_repo.go:82-121` | **model**, don't call (no `created_at` filter, no flag-supplied `high`) |
| scan projection + piece decode | `coord/db/audit_repo.go:57,73,85` | **reuse** |
| single segment | `GetAuditSegment` `audit_repo.go:101` | close, but drops `FileID`/`CreatedAt`/`Position` — new |
| segments by file | `file_repo.go:68` | different projection (`derived_key`/`nonce`) — new |
| segments by contract | *none* | new: `JOIN files ON files.id = segments.file_id WHERE files.contract_id = ?` (`idx_files_contract_id` covers it) |

Every query carries `AND pieces IS NOT NULL` (the predicate `GetAuditSegment` already uses) so
inline segments never enter the pipeline; keep the `seg.Inline()` guard in the service anyway.

Simpler than Storj: depin's `segments` PK is `id` alone (migration `000025`), so the cursor is a
single UUID — drop Storj's `(stream_id, position)` cursor and its `uuidBefore` hack.

`ListSegmentsByContract` defaults to `files.deleted_at IS NULL` with an `--include-deleted-files`
opt-in.

---

## The probe

```go
pieceNum, ok := findPieceNum(segment, alias)   // return ok; do NOT panic like Storj's verify.go:217
pieceSize := file.RedundancyScheme(segment.Redundancy).PieceSize(segment.EncryptedSize)
loc := order.PieceLocation{PieceNum: pieceNum, WorkerID: target.ID, WorkerAddress: target.Address}

privKey, limits, err := v.orders.CreateAuditGetOrderLimits(
    ctx, segment.RootPieceID, pieceSize, []order.PieceLocation{loc})
// on error: sleep OrderRetryThrottle, retry once, then outcome RETRY (Storj verify.go:158-172)

timedCtx, cancel := context.WithTimeout(ctx, v.config.PerPieceTimeout)
defer cancel()
data, err := v.fetcher.Fetch(timedCtx, *limits[0], privKey, 0, 1)

switch {
case err != nil:        return audit.ClassifyFetchError(err), nil
case len(data) != 1:    return placement.AuditFailure, nil   // fact #3 — must not be treated as success
default:                return placement.AuditSuccess, nil
}
```

`CreateAuditGetOrderLimits` (`coord/order/service.go:179`) is the right minting call. **Not**
`CreateAuditOrderLimits` (`:302`) — it resolves through `GetWorkersForAudit`, silently drops offline
and disqualified holders before minting, and returns a slice indexed by piece number with nil holes.
segment-verify exists precisely to probe and report those holders.

**Connection reuse.** Storj force-dials one client per node-batch. `dialFetcher.Fetch` dials per
call, but `dial.Dialer` pools per `"node:"+nodeURL` key, and exactly one goroutine owns each
node-batch, so per-key reuse gives the same behaviour with no code change. Set
`dialer.Pool = pool.New(pool.Options{Capacity: cfg.Concurrency, KeyCapacity: 2, IdleExpiration: 10*time.Minute})`
and `dialer.DialTimeout = vcfg.DialTimeout` (otherwise 20s from `NewDefaultDialer`).
Do **not** reach for `pool.WithForceDial` — depin's version (`internal/grpcutil/pool/pool.go:278-303`)
takes from the cache first and is not Storj's force-dial.

**Offline bail-out.** `dialFetcher` has no redial loop, so replace Storj's `maxDials` with a count of
*consecutive* `AuditOffline`/`AuditContained` outcomes inside a node-batch; at 2, return
`ErrWorkerOffline` along with `verifiedCount` and let `VerifyBatches` do the offline bookkeeping.
Remaining pieces stay in `Status.Retry` → retry CSV, exactly Storj's outcome.

---

## CLONE handling

The probe is byte-for-byte identical — a 1-byte read is scheme-agnostic. Three differences:

1. `pieceSize` — free, via `file.RedundancyScheme.PieceSize` (fact #6).
2. **Threshold**: RS needs `RequiredShares`; CLONE needs **1** (any copy reconstructs). This rule
   already exists at `coord/file/endpoint.go:1075-1079`. Encode once as
   `healthThreshold(scheme) int32`.
3. **`Check` clamping**: with `--service.check=3` against a 3-copy CLONE placement, Check equals all
   pieces. Storj's `CreateBatches` *skips* a segment when `len(Pieces) < Status.Retry` and logs an
   error; that is wrong for depin where narrow schemes are normal. Clamp
   `Retry = min(Check, len(Pieces))` and log at debug.

Verdict, both algorithms: `alive >= RepairShares` → `HEALTHY`; `threshold <= alive < RepairShares` →
`AT_RISK`; `alive < threshold` → `LOST`. Any `Retry > 0` downgrades to `INCONCLUSIVE` unless `Found`
already clears the bar.

**Stated limitation** (footer in `check` output and in the README): a 1-byte probe proves *presence*,
not *correctness*. It does not check the CLONE Merkle root and does not run the RS erasure
cross-check — `coord/audit` does both. Storj carries the same caveat.

---

## Worker resolution and offline handling

Resolve with an unfiltered slim projection —
`SELECT alias, id, address, last_seen, is_online, disqualified_at, deleted_at FROM workers`
(the analogue of Storj's `loadOnlineNodes`, once per run).

Not `GetWorkersForAudit` (`coord/db/worker_repo.go:114`, filters DQ/deleted/last_seen) — offline
holders must be probed and reported. `contact.Store.GetWorkersByAliases` (`worker_repo.go:85`) has
the right *semantics* (no liveness filter) but loads full `contact.Worker` rows including three
uptime ring-buffer BYTEAs and calls `ComputeScores()` per row; wasteful when resolving every alias
in the cluster.

**`ErrNoSuchWorker`** (Storj's `ErrNoSuchNode`): a piece whose alias has no `workers` row, or whose
row is disqualified/deleted → mark every segment in that batch `MarkNotFound()` immediately, no dial.
Faithful to Storj `process.go:98-107`.

**Offline TTL set**: port `nodeAliasExpiringSet` keyed on `int64`, **with an internal mutex**. Storj's
`Contains` (`nodealias.go:57-65`) deletes from the shared map on a value receiver without locking,
which is benign there only because `CreateBatches` and `VerifyBatches` never overlap. Note the
divergence in a comment so it does not read as an accidental deviation. Bookkeeping is otherwise
faithful: `verifiedCount == 0` → offline immediately; else `offlineCount[alias]++` until `MaxOffline`;
a successful batch decrements.

**Priority/ignore files**: depin worker ids are UUIDs, not base58. Accept either a canonical UUID
(`vo.NewIDFromString`) or a bare integer alias, one per line, `#` comments and blanks skipped,
log-and-skip unresolvable entries (Storj `service.go:229-233`). Flags
`--service.priority-workers-path` / `--service.ignore-workers-path`; keep Storj's rules that ignore
wins over priority and that priority workers bypass the throttle.

---

## Output

**`segments-not-found.csv` / `segments-retry.csv`**

```
segment id,segment number,file id,created at,algorithm,required,total,found,not found,retry
```

`segment id` ← Storj's `stream id`; `segment number` ← Storj's `position`. `file id` is new (lets an
operator jump straight to the affected file); `algorithm` is new and necessary — without it
`required` is un-interpretable across RS and CLONE. Column 0 is a UUID, so `read-csv` round-trips
these files.

**`problem-pieces.csv`**

```
segment id,segment number,created at,worker id,worker alias,piece number,outcome
```

`worker alias` added because depin's piece list is alias-keyed, which is what an operator has in hand.

**Outcome labels**: do not use `placement.AuditOutcome.String()` (`coord/placement/audit.go:78`),
which yields ambiguous lowercase `"failure"`. Map locally to Storj-compatible labels —
`AuditSuccess`→`SUCCESS`, `AuditFailure`→`NOT_FOUND`, `AuditOffline`→`NODE_OFFLINE`,
`AuditContained`→`TIMED_OUT`, `AuditUnknown`→`UNKNOWN_ERROR`, plus a **package-local** sentinel →
`RETRY` for "order minting failed twice". Do not add a constant to `placement.AuditOutcome` — that
enum is reputation-bearing and consumed by `coord/audit/reporter.go`.

CSV mechanics follow `internal/billing/csv.go:20-51`: package-level column slice, `Write…CSV(io.Writer, …)`,
`…Record(…) []string`, `Flush()` then `Error()`.

**`check` table** (stdout), plus `--json` and `--quiet`:

```
SEGMENT 018f3a71-9c2e-7b40-9f11-6d3b0a5c1e22   (file 3f2a…, segment #0, contract 88b1…)
  scheme     REED_SOLOMON 29/35/50/80   piece 141 KiB
  probed     80 of 80 pieces
  verdict    AT_RISK   found 33  lost 47  inconclusive 0   (repair at 35, required 29)

  PIECE  WORKER        ALIAS  ADDRESS             OUTCOME      TOOK
      3  018e…c41f       412  tcp:10.0.3.4:7777   SUCCESS       31ms
     17  018e…9b02        77  tcp:10.0.9.1:7777   NOT_FOUND     12ms
  note: existence probe only - bytes are not verified. Run `coord audit` for integrity.
```

---

## Config and wiring

`ServiceConfig` / `VerifierConfig` mirror Storj's, with two deliberate default changes:

| Flag | Storj | Here | Why |
|---|---|---|---|
| `--service.concurrency` | 1000 | **64** | see below |
| `--service.check` (on `check`) | 3 | **0** (all) | a verdict needs every piece |

Storj dials over plain TCP. depin dials through `connector.NewHybridConnector(tcp, p2p)`, and every
NAT'd worker is reached through one shared relay IP;
`internal/grpcutil/connector/p2p_connector.go:79-91` documents libp2p's per-IP connection cap
(`MaxConnsPerIP`, restored to 512 when zero). 1000 concurrent batches through one relay would exhaust
it and mass-report false offlines. Start at 64 and document raising it alongside
`server.resource-limits.max-conns-per-ip`.

Everything else keeps Storj's values: `BatchSize=10000`, `MaxOffline=2`,
`OfflineStatusCacheTime=30m`, `DialTimeout=2s`, `PerPieceTimeout=800ms`, `OrderRetryThrottle=50ms`,
`RequestThrottle=150ms`, plus `CreatedBefore`/`CreatedAfter` `DateFlag`s (portable now that
`segments.created_at` exists, migration `000042` — note it is **unindexed**, so those filters mean a
seq scan unless paired with a narrow `--low/--high`).

Bind `segmentVerifyCfg` **and** the leaf's own config to **every leaf command** — `cmd/coord/main.go:277-279`
documents that viper repopulates only the command that actually runs, so binding a parent leaves leaf
values zeroed.

Wiring is `coord/peer.go:207-243` `setupDialer()` minus the lifecycle group — no `setupServer()`, no
listeners:

```go
dbConn, _ := coorddb.ConnectDB(cfg.AppConfig.DB.PostgresDsn)   // NOT Migrate()
ident,  _ := cfg.Identity.Load()
revDB,  _ := revocation.OpenDB(ctx, revocationURL(cfg))
tlsOptions, _ := tlsopts.NewOptions(ident, cfg.Server.Config, revDB)
p2p, _ := connector.NewP2PConnectorFromIdentity(ident, cfg.Server.ResourceLimits)
dialer := dial.NewDefaultDialer(connector.NewHybridConnector(connector.NewDefaultTCPConnector(), p2p), tlsOptions)
ordersSvc, _ := order.NewService(log, signing.SignerFromFullIdentity(ident), nil, nil, cfg.Order)
fetcher := audit.NewDialFetcher(log, dialer)
```

**Revocation DB must default to `memory://`.** `Extensions.Revocation` defaults true
(`internal/peertls/extensions/extensions.go:69`) and `RevocationDBURL` to `bolt://$CONFDIR/revocations.db`;
`newBoltStore` takes an **exclusive flock** with a 1s timeout (`internal/revocation/store.go:44-49`).
A running coordinator holds it — so `segment-verify --config-dir <coord dir>` would die after 1s,
in exactly the scenario the tool exists for. Default to `memory://`, log one INFO line, and offer
`--revocation-dburl` to override. The security delta is nil: the tool takes no action on any worker.

---

## Build and release integration

- **`Makefile.build`**: add `build-segment-verify` / `release-segment-verify` to `.PHONY`;
  `SEGMENT_VERIFY_BIN=segment-verify`; append `segment-verify` to `COMPONENTS` (line 68) so
  `make version-info` lists it. Model the target on `build-worker-updater` (`:217`) and build by
  **package path** (`./cmd/segment-verify`), not a `*.go` glob — a file-list build disables Go's vcs
  stamping.
- **`scripts/component-version.sh`**: add
  `segment-verify) dirs="cmd/segment-verify coord/segmentverify coord/order coord/audit" ;;`.
  Mandatory — otherwise the release check fails with `unknown-component`.
- **`scripts/bake.sh`**: add `segment-verify` to `COMPONENTS`, else `release.Dockerfile` hard-fails
  with `no version for component`.
- **`release.docker-bake.hcl`**: append `./cmd/segment-verify` to the `components` string for
  **linux/amd64 and linux/arm64 only** — the other rows ship the operator-facing worker/keytool set,
  and this tool needs a coordinator DSN and identity.
- **`cmd/coord/Dockerfile`**: `COPY` the binary in next to keytool/relay-ping. It needs the coord's
  config dir, identity and DB, exactly like keytool.
- `version` subcommand comes free from `process.ExecCustomDebug`, which satisfies the end-to-end
  binary check in `scripts/release/check-release-binaries.sh`.

---

## Task order

| # | Task | Depends on |
|---|---|---|
| 1 | **Spike:** `segmentVerifyScan` embedding `auditSegmentScan` + `FileID`, one throwaway query against `coorddbtest` | — |
| 2 | `coord/db/segmentverify_repo.go` — 5 segment queries + 3 worker-target queries, with DB tests | 1 |
| 3 | `coord/segmentverify` skeleton: types, config, `aliasset.go` (with the mutex fix), `csv.go` + unit tests | — (parallel with 2) |
| 4 | `batch.go` — `CreateBatches` and friends + unit tests | 3 |
| 5 | `verify.go` — `NodeVerifier`, `rateLimiter` + unit tests against fake minter/fetcher | 3 |
| 6 | `process.go` + `service.go` — two-pass verify, `VerifyBatches`, worker cache, `RemoveDeleted` | 2,4,5 |
| 7 | `source.go` + `report.go` — four sources, verdict, table/JSON | 2,6 |
| 8 | `cmd/segment-verify/*` — cobra tree, per-leaf `process.Bind` | 6,7 |
| 9 | Manual smoke against `dev/` (`make setup-coord`, one upload, `check segment <id>`) | 8 |
| 10 | `internal/testplanet/segment_verify_test.go` + a `Coord.SegmentVerifyService()` helper | 8 |
| 11 | Build/release wiring (all five files above) | 8 |
| 12 | README + `make version-info` sanity check | 11 |

**Step 1 is the one genuine unknown**: gorm's `embedded` tag combined with an embedded type that
carries its own `TableName()`. Fallback if it misbehaves is to declare the 11 fields flat and call
the existing `auditPieces()` helper — still reusing the scanners, just not the struct. Resolve it
before writing step 2.

---

## Verification

**Unit** (`coord/segmentverify/*_test.go`, mirroring Storj's `batch_test.go`/`csv_test.go`/`nodealias_test.go`):
- `CreateBatches` — the 35%/50%/60% redistribution thresholds, priority-first ordering, priority
  batches exempt from redistribution, offline/ignored piece removal, and the depin `Retry = min(Check, len(Pieces))` clamp
- `workerAliasExpiringSet` — injected `nowFunc` for TTL expiry, plus concurrent `Contains`/`Add` under `-race`
- CSV round-trip `Write…CSV` → `SegmentCSVSource`, header handling, header-row skip
- outcome→label mapping, and the RS/CLONE verdict matrix across `alive ∈ [0, TotalShares]`
- `NodeVerifier` against a fake minter (model on the `fakeMinter` at `coord/audit/verifier_test.go:56-80`)
  and fake fetcher: 1 byte→`SUCCESS`; `codes.NotFound`→`NOT_FOUND`; `DeadlineExceeded`→`TIMED_OUT`;
  dial error→`NODE_OFFLINE`; **`(nil, []byte{})`→`NOT_FOUND`** (fact #3 — the regression test that matters);
  minting fails twice→`RETRY`
- `--low/--high` hex-prefix and canonical-UUID parsing; `DateFlag`; worker-file parsing

**DB** (`coord/db/segmentverify_repo_test.go` via `coorddbtest.Run`, as `file_repo_test.go` does):
keyset pagination covers the range with no gaps or duplicates across page boundaries; inline rows
never returned; `created_at` bounds; contract join scoping and `deleted_at`; `ExistingSegmentIDs`
after a delete; and `segmentVerifyScan` decoding `redundancy_scheme` and the RLE `pieces` blob
identically to `auditSegmentScan` for both RS and CLONE fixtures — this is the test that catches the
step-1 risk.

**Integration** (`internal/testplanet/segment_verify_test.go`; CI runs `go test -race -vet=off ./...`):
1. happy path — upload, verify, all `SUCCESS`, not-found CSV empty
2. lost piece — `DropCoordPieces` on one holder → exactly that piece `NOT_FOUND`
3. offline worker — `StopPeer` → `NODE_OFFLINE`, lands in **retry** not not-found (the important distinction)
4. gone worker — `MarkWorkerGone` → `ErrNoSuchWorker` marks not-found without dialing
5. CLONE — `CloneRedundancy`, drop 2 of 3 copies → `AT_RISK` not `LOST` (threshold 1)
6. retry pass — `--service.check=1` with one dead holder; second pass probes a *different* piece
7. inline segments skipped — zero probes
8. `RemoveDeleted` — delete the file mid-run, segment dropped from the CSV
9. round trip — run `range`, feed `segments-not-found.csv` back through `read-csv`, same result

**Live cluster only**: real relay/NAT reachability at concurrency (the `MaxConnsPerIP` ceiling), the
coordinator-side signing cost per probe, and `usedserials` churn — every probe burns a serial in the
target worker's table, bounded by `worker.storage.max-used-serials-size` (default 1 MB,
`worker/piecestore/service.go:53`) with random eviction when full. Not a functional break, but a real
reason to keep `--verify.request-throttle` on and concurrency modest. Retune defaults from
measurement, not from Storj's numbers.

---

## Out of scope

- Storj's `node-check` subcommand (subnet-duplicate / unvetted-node detection) — depin's placement
  and diversity model differs enough to need its own design.
- Any worker-side change, including an `Exists` RPC.
- Writing reputation or enqueuing repair.
- Byte-level integrity (that is `coord/audit`'s job).
