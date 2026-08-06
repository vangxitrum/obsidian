---
type: decision
tags: [depin, testing, testplanet, test-plan, qa]
created: 2026-08-04
updated: 2026-08-05
agent: main
---

Plan to turn `docs/test-plan.md` (159 TC rows, see [[depin-test-plan]]) into an
executing suite. Plan note:
`Projects/depin/plans/2026-08-04-testplanet-test-cases.md`.

**Update 2026-08-05 - Phase 2b, section 18 Deletion (DEL).** Deletion was absent
from both the plan AND `docs/test-plan.md`: 19 new TC-DEL rows + 15 tests. Facts
established while scoping it, all verified in the tree at `57dfcaa`:
- `DeleteFile` is metadata-only, hard, single-CTE, but the **RPC is not
  idempotent** even though the SQL is - `AuthorizeFileOwner`
  (`coord/file/service.go:120`) returns NotFound before the store's zero-rows
  no-op is reached. It also resolves the contract via the **live-only**
  `GetByID` (`:135`), so a file under a soft-deleted contract is NotFound too.
- `DeleteContract` mark-then-drain, `deleteBatchSize = 100`
  (`coord/storage/service.go:30`), resume path at `:113`.
- Neither touches worker pieces. Reclaim is GC-only; both GC halves default
  `Enabled=false`. Assert survival with
  `w.Storage.HashStoreBackend.SpaceUsage().UsedForPieces`.
- Delete-file's real usage invariant: the next tally writes an explicit **zero**
  row for the emptied contract (`fillContractTallies` zero-seed) so byte-hour
  integration stops billing. Historical byte-hours are untouched.
- `repair_queue` has **no FK to segments**, so a delete orphans queued jobs; the
  repairer drops them on lookup miss (`coord/repair/repairer/segments.go:100`).
- Package coverage is already thorough (6 storage unit + 3 contract-repo DB + 3
  file-service subtests + 1 file-repo DB), but the **endpoint/gRPC error-mapping
  layer has zero tests** and there is no e2e file-delete test at all. User chose
  a full planet-level mirror anyway, a deliberate exception to reason 4 below.
- `testplanet.Uplink` needs 4 new wrappers (`CreateContract`,
  `UploadToContract`, `DeleteFile`, `DeleteContract`) - `Upload` mints a fresh
  contract per call and hides its id. All 4 go-sdk methods exist in the pinned
  `v0.0.0-20260805041745-13b7d8dbe9a8`; no local `../go-sdk` replace needed.
- A `Coord.RepairOnce()` helper is required because `repairer.Worker.repairOne`
  is unexported and `Run` is an infinite loop - shared with the RPR commit.

**Three `docs/test-plan.md` rows name code that does not exist** (fix or they are
permanently un-implementable): TC-FS-11's "`task.go`" background processing
(`coord/file/task.go` is a GORM model + `Validate()`; the real chore is
`coord/file/expireddeletion`); TC-USG-06's `file_storage_finalized` table (real
tables: `contract_storage_tallies` / `worker_storage_tallies`); TC-FS-03's
multipart upload (**no multipart, no abort RPC, no zombie segments anywhere in
this repo**) - and section 1's commit currently budgets a test for it.

**hermes vault-write is still broken (2026-08-05).** `kr/claude-sonnet-4.5-agentic`
fails `HTTP 404: No active credentials for provider: kiro`, and the entire `kr/`
provider is now absent from the 9router `/v1/models` list (only `cx/gpt-5.6-*`
remain). `cx/gpt-5.6-sol` accepted the job but produced nothing in ~4 min. Wrote
the vault plan directly instead. Global CLAUDE.md still pins the dead `kr/` model.
See [[hermes-9router-provider]] in `_global`.

**Branch reality (verified 2026-08-04).** `feat/implement-testcases` was 104
commits BEHIND `origin/develop` and missing `coord/deposit`, `internal/billing`,
`pkg/nodetag`, `pkg/logship` entirely - it sits at `main` (9214408). The test
plan doc itself only exists on develop (`0bb5812 docs: test cases`). User chose
to **merge develop into** the branch rather than reset. Anyone touching test work
must be on develop or later.

**Coverage reality that the doc does NOT show.** ~35 of the 159 rows are already
green at package level, which is invisible from the document:
- `coord/deposit` has 33 tests (`service_test.go` fakeChain+fakeStore, `rate_test.go`,
  `sweeper_test.go`) covering nearly all of section 8 (DEP).
- `cmd/worker-updater/update_test.go` (7) + `internal/version/checker` (6) +
  `versioncontrol/api` (8) cover nearly all of section 11 (UPD).
- `coord/relay/peer_test.go` (4) + `worker/relay` (11) cover the bootstrap half
  of section 13 (RLY).
So of the three harness extensions the user picked, only the **relay peer** has
real payoff (nothing exercises the data path over a relayed worker: FS-13,
RLY-06, AUD-05). versioncontrol-in-planet and the deposit chain fake were scoped
down to one wiring test each.

**Why testplanet cannot cover ~25 rows** - five structural properties, not gaps:
1. One process, goroutine peers under one errgroup. No exec/PID/argv. Kills
   UPD-09/10/11 (install, per-OS restart, crash-loop).
2. All peers on 127.0.0.1 and `cfg.Overlay.DistinctNetwork=false` in
   `testplanet/coord.go`. One network, no country. Kills OVL-02, PLC-04,
   WINFO-05/06.
3. No CLI layer - it calls `coord.NewPeer`/`worker.NewPeer`/`uplinksdk.New`
   directly, never parses flags. Kills REG-01..09, WTAG-05 (keytool).
4. A planet boot = Postgres schema + migration + N peers; absurd for a pure
   function. TAG-02..05, RPR-02/04/06, RWD-08, PAY-06/09, DEP-03/04/05/07,
   UPD-02/03/04.
5. No non-Go systems. MON-07/08 are Grafana JSON dashboards.
Plus WTAG-08 and AUD-08 are absence/parity assertions no test can pass or fail,
and RLY-08 (double NAT) needs a real NAT device rewriting source ports.

**Enablers found.** Every heavy subsystem is config-gated and reachable from
`Reconfigure.Coord`: `Config.Audit.Enabled`, `Repair.Enabled`,
`GC.BloomFilter.Enabled`, `GC.Sender.Enabled`, `Deposit.Enabled`. No harness
surgery needed for AUD/RPR/GC. `deposit.NewService(log, config, chain, store,
rates)` is directly injectable, so a fake chain needs a testplanet helper next to
the existing `AuditVerifier()` pattern - no production seam. Relay advertisement
is `coord.Config.Contact.RelayAddrs` (`coord/contact/service.go:56`), so a planet
relay peer just needs constructing before the coords.

**Bug found while exploring.** `TestEdgeserverMetadataHeaders` asserts an empty
`Cache-Control` that the committed handler always sets - FS-10 reads as covered
but is a false green. Same finding as [[edgeserver-metadata-headers]]-era work;
see also the standalone memory `project_edgeserver_metadata_test_stale`.

**Skip trap.** `testplanet.Run` skips silently without `DEPIN_TEST_POSTGRES`
(or `-postgres-test-db`), and a fully-skipped suite exits 0. Never read a green
run as coverage without counting executed tests.
