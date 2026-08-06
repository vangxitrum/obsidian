---
type: decision
tags: [depin, segment-verify, audit, orders, storj-parity]
created: 2026-08-06
agent: main
---

Plan for a standalone `cmd/segment-verify` tool (manual/on-demand segment health checking), ported
from Storj `cmd/tools/segment-verify`. Branch `feat/segment-verifier`. Plan lives at
`Projects/depin/plans/2026-08-06-segment-verify.md`; nothing implemented yet as of 2026-08-06.

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
`Planet.PieceHolders` (`:135`), `Coord.AuditSegmentForFile` (`:94`), `Coord.MarkWorkerGone` (`:313`),
`Planet.StopPeer` (`planet.go:340`), `testplanet.CloneRedundancy` / `DefaultRedundancy`
(`planet.go:65-86`). This is why the library goes in `coord/segmentverify/` and not `package main` -
`internal/` cannot import a main package, which is how Storj gets away with a single-package tool.

**Highest-uncertainty implementation item:** gorm `embedded` tag on a struct embedding
`auditSegmentScan` (which carries its own `TableName()`), to add `FileID`. Spike it first; fallback is
a flat 11-field struct still reusing the `auditPieces()` helper and the `file.RedundancyScheme` /
`file.Pieces` scanners.

Related: [[worker-hashstore-pieces-endpoint]], [[coord-audit-metrics-instrumentation]],
[[coord-repair-metrics-instrumentation]], [[depin-gitignore-allowlist-gotcha]].
