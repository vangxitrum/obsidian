# Plan: Implement `docs/test-plan.md` test cases with testplanet

## Context

`docs/test-plan.md` (committed on `origin/develop` as `0bb5812`) is a 159-row standing
regression list covering 17 feature areas, every row still marked `Not Run`. It is a
manual document: nothing in the repo executes it, and nothing links a TC-ID to code.

The goal is to turn that document into an executing suite. `internal/testplanet` is the
right vehicle: it boots real coordinator, worker, and uplink peers in one Go test process
over loopback TCP against a real per-coord Postgres schema, so a test drives genuine gRPC,
real signing, and real chores rather than mocks. It already holds 17 test files / ~24
tests covering roughly 24 of the 159 rows.

Outcome: every TC row that is genuinely a multi-peer end-to-end behaviour becomes a named
Go test, `docs/test-plan.md` gains a traceability column linking each row to its test, and
the rows testplanet structurally cannot serve are reclassified (with reasons) to package
tests or Manual.

## Decisions taken

- **Base**: merge `origin/develop` into `feat/implement-testcases` (branch is 104 commits
  behind and lacks `coord/deposit`, `internal/billing`, `pkg/nodetag`, `pkg/logship`).
- **Scope**: testplanet-feasible rows only. Non-feasible rows are documented, not silently
  dropped. See "Why testplanet cannot cover the rest".
- **Harness**: build all three extensions (relay peer, versioncontrol peer, deposit chain
  fake). See the cost/benefit note under Phase 1 - two of the three have far less payoff
  than they appear, because package tests already cover most of those sections.
- **Delivery**: one branch, one commit per section, suite green at every commit.

## Why testplanet cannot cover the rest

Five structural properties of the harness decide this. They are properties, not gaps to
patch.

1. **One process, goroutine peers.** `planet.go` runs every peer as a goroutine under one
   `errgroup`, started via `peer.Run(ctx)`. There is no `exec`, no PID, no argv, no
   stdout. So anything that installs a binary, kills a process, or observes a restart is
   out: `TC-UPD-09/10/11` (install + restart, per-OS restart paths, crash-on-start
   bricking). `cmd/worker-updater/update_test.go` already covers this with a real
   subprocess harness.
2. **Every peer is on `127.0.0.1`, and `Overlay.DistinctNetwork` is forced off**
   (`coord.go`). There is exactly one network, one `last_net`, no country. So network
   diversity and server-derived geo cannot be exercised through real planet workers:
   `TC-OVL-02`, `TC-PLC-04` (distinct /24), `TC-WINFO-05/06` (LastNet, ResolvedIP,
   CountryCode, EgressNets). These need synthetic `workers` rows, which is a
   `coord/overlay` / `coord/contact` package test, not a planet.
3. **No CLI layer.** testplanet constructs `coord.NewPeer` / `worker.NewPeer` /
   `uplinksdk.New` directly. It never parses flags or runs a `cmd/`. So the whole
   `keytool` lifecycle is out: `TC-REG-01..09` (key lifecycle, `create`, `id`, `sign-ca`,
   `authorize`, `sign-server` + auth tokens, `new-piece`), plus `TC-WTAG-05`
   (`keytool sign-tags`). These belong in `worker/pkg/keytool` package tests with a
   bufconn sign-server.
4. **A planet boot costs a Postgres schema + migration + N peers.** For a pure function
   that is pure overhead and buries the unit under integration noise. Out on those
   grounds: `TC-TAG-02/03/04/05` (`coord/tags` validation), `TC-RPR-02/04/06`
   (redundancy/health math, `ClassifySegmentPieces`, priority ordering), `TC-RWD-08`
   (`internal/period` boundaries), `TC-PAY-06/09` (invoice CSV round trip,
   `Policy.CheckMargin`), `TC-DEP-03/04/05/07` (rate sources, `usdValue`),
   `TC-UPD-02/03/04` (rollout cursor, version filter, checker compare). Most already have
   package tests.
5. **No non-Go systems.** `TC-MON-07/08` assert Grafana panels render real data against a
   live Prometheus. Those are JSON dashboards under `monitoring/`, not code. Best
   automatable form is a JSON-lint + PromQL-parse check; the rendering claim stays Manual.

Two rows are assertions about absence or parity rather than behaviour, so no test can
meaningfully pass or fail them as written: `TC-WTAG-08` ("confirm tags are still NOT
consumed by placement") and `TC-AUD-08` ("parity after the rangedloop refactor"). Both get
marked Manual with a note.

One row cannot be simulated in-process at all: `TC-RLY-08` (double-NAT worker that can
never hole-punch). The failure mode *is* a real NAT device rewriting the source port; a
loopback planet has no such device. Fleet/manual only.

## Coverage math

| | rows |
|---|---|
| Already covered by existing testplanet tests | ~24 |
| Already covered by existing package tests | ~35 |
| New testplanet tests (this plan) | ~90 |
| New deletion tests (section 18, added later) | ~15 |
| Package tests, listed as follow-up | ~19 |
| Manual / not automatable | ~6 |

The "already covered by package tests" number is large and was not obvious from the
document: `coord/deposit` (33 tests across service/rate/sweeper), `cmd/worker-updater`
(7), `internal/version/checker` (6), `versioncontrol/api` (8), `coord/relay` (4),
`worker/relay` (11) between them already satisfy most of sections 8, 11, and 13.

---

## Phase 0 - base and traceability

**Commit: `chore(test): merge develop, add TC traceability to test plan`**

1. `git merge origin/develop` into `feat/implement-testcases`. The branch has no unique
   commits, so this is a fast-forward in practice.
2. Add a `Test` column to every table in `docs/test-plan.md` holding the Go test name
   (e.g. `TestUpload`) or one of `pkg:<package>` / `Manual`. Pre-fill the ~24 rows the
   existing tests already satisfy and the ~35 rows package tests satisfy.
3. Add a short header paragraph explaining the three statuses and pointing at
   `internal/testplanet/README.md`.

Each later commit updates its section's `Test` and `Status` columns in the same commit as
the tests. That keeps the document honest at every point in history.

---

## Phase 1 - harness extensions

Three commits. All three are additive to `internal/testplanet`; only the relay one touches
`planet.go` ordering.

### 1a. `feat(testplanet): boot relay peers and behind-NAT workers`

Highest-value of the three. Nothing today exercises the real data path over a relayed
worker, which is where several past production incidents lived.

- Add `RelayCount int` to `testplanet.Config` and `Relays []*Relay` to `Planet`.
- New `internal/testplanet/relay.go`: wraps `coord/relay.NewPeer(log, ident, config)`,
  implements the `Peer` interface (`Label`/`ID`/`Addr`/`Run`/`Close`), exposes
  `P2PAddrInfo()`.
- Ordering in `createPeers`: relays are constructed and started **first**, then coords,
  then workers. Coords need the relay multiaddrs at construction time to fill
  `cfg.Contact.RelayAddrs` (`coord/contact/service.go:56`), which is what
  `Endpoint.GetRelayAddrs` serves; workers already require serving coords because a
  behind-NAT worker resolves the relay fleet synchronously in `worker.NewPeer` (existing
  comment in `planet.go`).
- Add `Config.WorkerBehindNAT bool`: when set, `worker.go` stops forcing
  `cfg.Server.HavePublicAddress = true` and instead leaves `P2PAddress` populated so the
  worker builds a libp2p host and reserves on the relay via AutoRelay. Reuse the host
  wiring already proven in `coord/relay/peer_test.go`'s `newAutoRelayWorkerHost` and
  `workerrelay.WaitForCircuitAddr`.
- Helper `(*Planet) WaitForReservations(ctx, n)` mirroring the existing
  `waitForOnlineWorkers` idiom.

### 1b. `feat(testplanet): serve versioncontrol in-planet`

Low payoff, kept because it closes the one wiring gap package tests cannot: a real worker
reporting a new version at check-in.

- New `internal/testplanet/versioncontrol.go`: run `versioncontrol/api.NewServer(cfg, log,
  handler)` on an `httptest.Server` (a gin handler, so no peer lifecycle needed), and add
  `Config.VersionControl *VersionControlConfig` to enable it.
- Expose its URL so a test can point `internal/version/checker` at it.
- Do **not** attempt to drive `cmd/worker-updater`'s install/restart from a planet; see
  reason 1 above. `update_test.go` owns that.

### 1c. `feat(testplanet): fake deposit chain helper`

Low payoff for the same reason: `coord/deposit/service_test.go` already has a `fakeChain`
+ `fakeStore` covering scanning, watermarking, idempotency, address-encoding, and
confirmations. What is missing is proof that the coord peer's wiring credits into the real
coord DB.

- `setupDepositWatcher` in `coord/peer.go` hard-constructs `deposit.NewEVMChain` from
  config, so a fake cannot be injected through `Reconfigure.Coord`. Rather than add a seam
  to production code, add a helper next to the existing `AuditVerifier()` pattern in
  `internal/testplanet/helpers.go`:
  `func (c *Coord) DepositWatcher(chain deposit.Chain, rates deposit.RateSource) *deposit.Service`
  building the service against `coorddb.NewBillingRepository(c.DB.GetDB())` - the coord's
  real store.
- Add a small exported `testplanet.FakeChain` (canned blocks + tip, ~40 lines, mirroring
  the private one in `service_test.go`).

---

## Phase 2 - one commit per feature section

Each commit adds `internal/testplanet/<section>_test.go`, names each test after its
behaviour, tags the TC-ID in the doc comment, and updates the section's table rows. Order
is by dependency (harness-free sections first, relay-dependent last).

Sections, with what is new. Existing tests named in parentheses are extended, not
duplicated.

| # | Section | New testplanet tests | Notes |
|---|---|---|---|
| 1 | **File Storage (FS)** | 8 | inline-threshold upload (FS-02 CLONE half is `TestUploadClone`), multipart duplicate-part + bad-checksum (FS-03), segment/stream id resolution (FS-04), ticket wrong-owner/tampered/expired (FS-08), ETag 304 + gzip + canonical route + CORS (FS-09), cleanup chore after delete/abort (FS-11). Complete FS-07 with out-of-bounds and overlapping ranges (`TestEdgeRangedDownload` has closed/open/suffix). **Fix `TestEdgeserverMetadataHeaders` first**: it asserts an empty `Cache-Control` the committed handler always sets, so FS-10 is currently a false green. |
| 17 | **Contract & File Tags (TAG)** | 3 | contract tags at the limit persist (TAG-01), unknown-owner rejected before tag validation (TAG-06), same rules on file tags (TAG-07). TAG-08 is `TestContractTagUsage`. |
| 5 | **Usage (USG)** | 5 | worker tally (USG-02), paid-vs-unsettled egress (USG-05), `file_storage_finalized` survives delete (USG-06), usage reconciles with invoice (USG-07) and with compensation (USG-08). |
| 15 | **Placement (PLC)** | 5 | placement CRUD (PLC-01), country + upload filters (PLC-02), RS vs CLONE `ECParameters` (PLC-03), unknown `placement_id` rejected (PLC-05), `GetAvailablePlacements` (PLC-06). |
| 12 | **Overlay / Selection (OVL)** | 6 | upload cache rebuild + gauges (OVL-01), download cache includes suspended (OVL-03), stale/empty snapshot behaviour (OVL-04), disqualify vs suspend eligibility (OVL-05), `SelectWorkers` filters (OVL-06), short pool `HasSufficientWorkers` (OVL-07). |
| 4 | **Worker Info (WINFO)** | 7 | hardware/OS/version persisted (WINFO-01), empty version not stored (WINFO-02), uptime ring buffers across gaps (WINFO-03), `ComputeScores` (WINFO-04), `SeenWithin` boundary (WINFO-07), `SelectedWorker` projection (WINFO-08), hashstore-pieces paging (WINFO-09), idle-connection `too_many_pings` (WINFO-10, long-running). |
| 3 | **Worker Tag (WTAG)** | 6 | self-signed tags at check-in (WTAG-01), unknown signer rejected (WTAG-02), authority-signed vs self-signed `trusted_node` (WTAG-03), tags survive restart (WTAG-04), `/worker-tags/` valid/malformed/unknown (WTAG-06), tampered/expired/replayed blob (WTAG-07). |
| 2 | **Payment (PAY)** | 5 | credit and balances against the real ledger (PAY-03/04), free-egress allowance (PAY-08), exhausted-balance behaviour (PAY-10), `record-period` idempotency (PAY-11). |
| 6 | **Reward (RWD)** | 3 | progressive SDK order signing on a large transfer (RWD-03), one-off payments (RWD-06), statement matches recorded payments (RWD-09). |
| 9 | **Audit (AUD)** | 4 | full rangedloop observer pass (AUD-03), gated-off scores only (AUD-06), disqualification with gating on (AUD-07), and AUD-05 deferred to the relay commit. Enable via `Reconfigure.Coord` setting `Config.Audit.Enabled`. |
| 10 | **Repair (RPR)** | 8 | checker finds under-redundant segments (RPR-01), insert-buffer batching + dedupe (RPR-05), reconstruct from survivors (RPR-07), repaired pieces uploaded to new workers (RPR-08), kill N holders then repair end to end (RPR-09), out-of-placement forces repair (RPR-10), excluded-country flagged independently (RPR-11), healthy segment queues nothing (RPR-12). Enable via `Config.Repair.Enabled`. |
| 14 | **Garbage Collection (GC)** | 7 | bloom filter per worker, inline skipped (GC-01), metrics + bundle upload (GC-03), sender dispatch then no-op (GC-04), missing contact row skipped (GC-05), duplicate Retain idempotent (GC-06), success/fail counters (GC-07), piece reclaimed only after GC (GC-08). Enable via `Config.GC.BloomFilter.Enabled` + `Config.GC.Sender.Enabled`. |
| 16 | **Monitoring (MON)** | 3 | coord push cycle to an httptest remote_write receiver, cumulative not delta (MON-01), disabled guard returns immediately (MON-02), unreachable endpoint degrades gracefully (MON-03). MON-04/05/06 go to `pkg/logship` package tests. |
| 8 | **Deposit (DEP)** | 2 | watcher credits through the coord's real store into a queryable balance (DEP-01), watermark resume across a peer restart (DEP-02 wiring half). Everything else is already in `coord/deposit`. |
| 11 | **Worker Auto-Update (UPD)** | 2 | versioncontrol served in-planet returns the rollout-correct version to a real checker (UPD-01 wiring), worker reports its new version at the next check-in (UPD-12). |
| 7 | **Register (REG)** | 2 | fresh identity onboards and becomes selectable for upload (REG-11), peertls rejects a revoked / low-POW / non-whitelisted leaf at a real planet handshake (REG-10). |
| 13 | **Relay / P2P (RLY)** | 6 | relay bootstrap resolves and caches (RLY-01), all-coords-unreachable falls back to cache (RLY-02), fleet drift detected at check-in (RLY-03), idle keepalive both directions (RLY-04, long-running), concurrent piece downloads over one limited circuit (RLY-06), audit dials relayed workers (RLY-07 = AUD-05). Plus **FS-13** (relay-only upload/download) lands here. |
| 18 | **Deletion (DEL)** | 15 | 9 new file-delete tests + 5 new contract-delete tests + upgrading the existing `TestDeleteContract` fixture to a real RS file. Full mirror at planet level (deliberate exception to reason 4). Needs 4 new `Uplink` wrappers and a `Coord.RepairOnce` helper. See "Phase 2b - Deletion" below. |

---

## Phase 2b - Deletion (section 18)

Deletion was missing from this plan entirely. The repo has two delete RPCs, both shipped and
merged (`57dfcaa`), and `docs/test-plan.md` has **no rows for either**:

- `FileService.DeleteFile` - metadata-only hard delete of one file + its segments
  (`coord/file/endpoint.go:673` -> `coord/file/service.go:298` -> `coord/db/file_repo.go:102`).
- `StorageService.DeleteContract` - soft-delete the contract row, then hard-delete its
  files/segments in 100-row batches (`coord/storage/endpoint.go:54` ->
  `coord/storage/service.go:116` -> `coord/db/file_repo.go:196`).

Coverage today:

| Layer | Delete-contract | Delete-file |
|---|---|---|
| Unit (`coord/storage/service_test.go`) | 6 tests | - |
| Unit (`coord/file/service_test.go:314`) | - | `TestServiceDelete` (3 subtests) |
| DB (`coord/db/contract_repo_test.go`, `file_repo_test.go`) | 3 tests | 1 test |
| Endpoint / gRPC error mapping | none | **none** |
| testplanet e2e | `TestDeleteContract` (1, inline file only) | **none** |

Scope decision: **full mirror at planet level** - every case is re-asserted end to end even
where a package test already covers the logic, because the planet exercises the real mTLS
identity, the real SDK client, real Postgres, and real workers. This is a deliberate
exception to reason 4 under "Why testplanet cannot cover the rest".

### Doc corrections (same commit)

Three rows in `docs/test-plan.md` describe code that is not in the repo. Fix them or they
become permanently un-implementable tests:

1. **TC-FS-11** (`docs/test-plan.md:23`) says cleanup runs via "background task processing
   (`task.go`)". `coord/file/task.go` is a GORM model plus `Validate()` - there is no task
   chore. The real abandoned-upload cleanup is `coord/file/expireddeletion` (hard-deletes
   `PENDING` files older than `Config.File.FileExpiration`, wired at `coord/peer.go:1013`,
   exposed as `coord.File.ExpiredDeletion`). Reword the row to name that chore.
2. **TC-USG-06** (`docs/test-plan.md:80`) expects "`file_storage_finalized` usage survives
   the delete". No such table exists in code or in `coord/db/migrations/`. The real usage
   tables are `contract_storage_tallies` / `worker_storage_tallies`. Reword to the actual
   invariant, which TC-DEL-09 below covers properly.
3. **TC-FS-03** (`docs/test-plan.md:15`) specs a multipart upload with duplicate parts and a
   bad checksum. There is **no multipart upload, no abort RPC, and no zombie-segment concept
   in this repo** (`grep -i zombie` -> 0 hits; every `multipart`/`abort` hit is unrelated).
   Section 1's commit budgets a test for it. Either drop the row or restate it as
   multi-segment upload, which does exist.

### New rows: `## 18. Deletion (DEL)`

Added after section 17, same `ID | Type | Steps | Expected | Status` table shape.

**File deletion**

| ID | Type | Case |
|---|---|---|
| TC-DEL-01 | Positive | Upload an RS file through the SDK, `DeleteFile`, download again. Rows in `files`/`segments` gone; download fails NotFound. |
| TC-DEL-02 | Positive | Same for an inline file (below the inline threshold) - the inline payload row goes with it. |
| TC-DEL-03 | Positive | Multi-segment file: every segment row gone, a sibling file in the same contract untouched. (The plan originally also named `segment_pieces` / `segment_piece_uploads`; those tables were dropped by later migrations - pieces live in a `bytea` column on `segments`.) |
| TC-DEL-04 | Negative | A second uplink identity calls `DeleteFile` on someone else's file. `PermissionDenied`; the file is still downloadable by its owner. |
| TC-DEL-05 | Negative | Delete an unknown file id, and delete the same file twice. Both `NotFound` - `AuthorizeFileOwner` runs before the idempotent store call, so the RPC is *not* idempotent even though the SQL is. |
| TC-DEL-06 | Negative | Delete a file whose contract has already been soft-deleted. `NotFound`, because `AuthorizeFileOwner` resolves the contract through the live-only `GetByID` (`coord/file/service.go:135`). |
| TC-DEL-07 | Positive | Delete a `PENDING` file mid-upload (`CreateFile` + `BeginSegment`, no `CommitFile`). Rows gone; a later `CommitFile` on that stream fails. |
| TC-DEL-08 | Regression | After `DeleteFile`, the holder workers still report the piece bytes. Proves the delete is metadata-only; reclamation is GC's job (pairs with TC-GC-08). |
| TC-DEL-09 | Regression | Tally the contract, delete its only file, tally again. Historical byte-hours are unchanged, and the next tally writes an explicit **zero** row for the emptied contract so byte-hour integration stops billing it (`fillContractTallies` zero-seed, `coord/accounting/tally/tally.go`). This is the real form of TC-USG-06. |
| TC-DEL-10 | Regression | Delete a file that is already queued in `repair_queue`. The repairer drops the orphan job on segment-lookup miss and does not crash - `repair_queue` has no FK to `segments` (`coord/repair/repairer/segments.go:100`). |

**Contract deletion**

| ID | Type | Case |
|---|---|---|
| TC-DEL-11 | Positive | `DeleteContract force=false` on an empty contract. Soft-deleted (`deleted_at` set, `status=DELETED`), 0 files reported. |
| TC-DEL-12 | Negative | `force=false` on a contract holding a file. `FailedPrecondition`, and **nothing changed** - contract still live, file still downloadable. |
| TC-DEL-13 | Positive | `force=true` on a contract holding a real RS (non-inline) file. Contract soft-deleted, files and segments hard-deleted, `DeletedFileCount` correct. |
| TC-DEL-14 | Boundary | Contract holding more than `deleteBatchSize` (100) files. The drain loop runs multiple batches and removes all of them. |
| TC-DEL-15 | Negative | A second uplink identity calls `DeleteContract`. `PermissionDenied` over real mTLS. |
| TC-DEL-16 | Negative | Unknown contract id. `NotFound`. |
| TC-DEL-17 | Positive | Re-invoke `force=true` on an already-soft-deleted contract that still has residual files (a crashed drain). It resumes and finishes, no error. |
| TC-DEL-18 | Regression | Billing/usage history survives: `GetClientUsage` and `QueryContractPeriodUsage` return the same values before and after; `CreateFile` against the deleted contract fails. |
| TC-DEL-19 | Regression | After the force delete, holder workers still hold the drained files' pieces - contract-scoped form of TC-DEL-08. |

TC-DEL-08/19 assert only the "still held" half. The "reclaimed after a GC pass" half stays
TC-GC-08 in the section-14 commit; cross-reference it rather than duplicating the GC harness.

### Harness additions

`internal/testplanet/uplink.go` - `Uplink.Upload` mints a fresh contract per call and does
not surface its id (`uplink.go:47-55`), so no existing helper can drive a delete test. Add
four thin wrappers over the already-available go-sdk methods:

```go
func (client *Uplink) CreateContract(ctx context.Context) (vo.UUID, error)
func (client *Uplink) UploadToContract(ctx context.Context, contractID vo.UUID, data []byte) (vo.UUID, error)
func (client *Uplink) DeleteFile(ctx context.Context, fileID vo.UUID) error
func (client *Uplink) DeleteContract(ctx context.Context, contractID vo.UUID, force bool) (int64, error)
```

They wrap `Client.CreateContractWithPlacement` (go-sdk `placement.go:32`),
`Client.UploadFile` (`upload.go:104`), `Client.DeleteFile` (`upload.go:247`), and
`Client.DeleteContract` (`placement.go:53`) - all present in the pinned
`go-sdk v0.0.0-20260805041745-13b7d8dbe9a8`, so no SDK change and no `../go-sdk` local
replace is needed. Refactor `Upload` to `CreateContract` + `UploadToContract` so there is
one upload path, not two.

`internal/testplanet/helpers.go` - one helper for TC-DEL-10, next to the existing
`AuditVerifier()` pattern:

```go
// RepairOnce drains at most one job from the repair queue.
func (c *Coord) RepairOnce(ctx context.Context) error
```

`repairer.Worker.repairOne` is unexported and `Run` is an infinite loop
(`coord/repair/repairer/repairer.go:118,163`), so a test cannot drive a single repair today.
This helper is shared with the section-10 (RPR) commit - **land TC-DEL-10 with or after that
commit**, not before.

No `planet.go` change.

### Production changes this actually required (found during execution)

The plan said "no production-code change". Two were unavoidable, both pre-existing defects
the new tests exposed rather than caused:

1. **Duplicate migration `000033`** (blocker). `origin/develop` carries both
   `000033_worker_tags` (2026-07-24) and `000033_worker_version` (2026-07-31); golang-migrate
   refuses the set with `duplicate migration file: 000033_worker_version.up.sql`, so **every
   testplanet test on develop fails to boot**. Resolved by renumbering `worker_version` to
   `000040` (user's call). Caveat accepted: a DB whose lineage applied `worker_version` as 33
   (the local `coord-db` is one - version 38, has the `version` column, no `worker_tags`
   table) will now skip `000033_worker_tags` permanently and needs that migration applied by
   hand.
2. **`vo.GrpcError` had no `Forbidden` case** (`internal/vo/errors.go`), so every
   authorization failure routed through it surfaced as `Internal` instead of
   `PermissionDenied`. Its HTTP twin `GetHttpStatusCodeFromError` maps `Forbidden` to 403,
   and `coord/storage/endpoint.go` hand-patched the same mapping locally - which is exactly
   why `TestDeleteContractRejectsNonOwner` passed while `TestDeleteFileRejectsNonOwner`
   failed with `code = Internal desc = file: forbidden`. Fixed centrally with one case, so
   every endpoint benefits rather than just `DeleteFile`. No test asserted the old behaviour.

This is the payoff of the full-mirror scope decision: the endpoint / gRPC error-mapping layer
had zero tests, and it was the layer that was actually broken.

### Commit

Ordered after section 1 (File Storage) and section 5 (Usage), since TC-DEL-09 reuses the
tally fixtures those commits establish.

**`test(testplanet): delete file and delete contract end to end`**

New file `internal/testplanet/delete_file_test.go`:

| Test | Covers |
|---|---|
| `TestDeleteFileRemote` | DEL-01, DEL-08 |
| `TestDeleteFileInline` | DEL-02 |
| `TestDeleteFileMultiSegment` | DEL-03 |
| `TestDeleteFileRejectsNonOwner` | DEL-04 (`UplinkCount: 2`) |
| `TestDeleteFileUnknownAndRepeatNotFound` | DEL-05 |
| `TestDeleteFileUnderDeletedContractNotFound` | DEL-06 |
| `TestDeleteFilePending` | DEL-07 |
| `TestDeleteFileUsageSurvivesAndTallyZeroes` | DEL-09 |
| `TestDeleteFileOrphansRepairQueueJob` | DEL-10 (needs `RepairOnce`) |

Extend `internal/testplanet/delete_contract_test.go`:

| Test | Covers | Note |
|---|---|---|
| `TestDeleteContract` | DEL-12, DEL-13, DEL-18, DEL-19 | Exists. Upgrade its fixture from `MakeInlineSegment` to a real RS upload so DEL-19 rides along, and add the "nothing changed" assertions to the `force=false` step. |
| `TestDeleteContractEmptyWithoutForce` | DEL-11 | new |
| `TestDeleteContractDrainsAcrossBatches` | DEL-14 | new; 101 inline files via `coord.File.Endpoint` so no worker traffic |
| `TestDeleteContractRejectsNonOwner` | DEL-15 | new |
| `TestDeleteContractUnknownNotFound` | DEL-16 | new |
| `TestDeleteContractResumesDrain` | DEL-17 | new |

Reuse the package's existing fixtures rather than adding new ones: `testRS`
(`upload_test.go:18`), `remoteSize` (`upload_test.go:28`), `waitForOnlineWorkers`
(`upload_test.go:106`), `tagPeerContext` (`tag_usage_test.go:30`),
`findContractPeriodUsage` (`delete_contract_test.go:231`), `coord.AuditSegmentForFile` +
`planet.PieceHolders` (`helpers.go:45,86`), and
`w.Storage.HashStoreBackend.SpaceUsage().UsedForPieces` for the piece-survival assertions
(same idiom as `worker_stored_bytes_test.go:78`).

Two cases cannot be produced through the public API and need a documented DB nudge:

- **DEL-17** - a soft-deleted contract with residual files only happens after a crashed
  drain. Create the contract with files, stamp `deleted_at` + `status='DELETED'` directly,
  then call `DeleteContract(force=true)` and assert it finishes the drain.
- **DEL-06** - same nudge, or reach it naturally by deleting the contract with `force=false`
  while it is empty and then inserting a file row under it.

Update the section-18 `Test`/`Status` columns in the same commit, per the Phase 0
traceability rule.

### Verification for this commit

```
go build ./...
go test ./internal/testplanet/ -run 'TestDeleteFile|TestDeleteContract' -count=1 -v
go test ./coord/... -count=1
go test ./internal/testplanet/ -race -run 'TestDelete' -count=1
```

Expect 15 `--- PASS` lines from the `-run` invocation (9 new file tests + 5 new contract
tests + the existing `TestDeleteContract`). A `SKIP` means Postgres was not configured, not
that the case passed. For DEL-08/DEL-19, assert the holder set is non-empty *before*
deleting, so a selection failure does not silently pass as "no pieces to check".

### Result (executed 2026-08-05)

`ok aioz-depin/internal/testplanet 62.475s` - all 15 delete tests pass. Full suite
(`go test ./internal/testplanet/ -count=1`, 296s) is green except `TestEdgeserverMetadataHeaders`,
which fails identically with the delete work stashed (`Should be empty, but was
private, max-age=0, immutable`) - the already-documented false green. `coord/db`,
`coord/file`, `coord/storage` all pass. `internal/vo` has one unrelated pre-existing failure,
`TestPieceIDScanNullAndEmpty`, also confirmed failing without these changes.

Two traps worth remembering:

- `go build ./...` cannot pass on this branch at all: `cmd/uplink` imports the `uplink/`
  package that commit `63564ce` ("refactor: remove keytool") deleted. Build
  `./internal/... ./coord/... ./worker/...` instead.
- Writing the "nothing changed" download into `TestDeleteContract`'s force=false step broke
  the later `preUsage.Egress == postUsage.Egress` assertion, because that download is real
  egress recorded after the snapshot. The rejection step now runs *before* the usage
  snapshot.

---

## Phase 3 - follow-up list (not implemented here)

A final commit adds a short `## Not covered by testplanet` appendix to `docs/test-plan.md`
listing the ~19 package-test rows with their target package, and the ~6 Manual rows with
the reason. This is the deliverable that keeps the document honest rather than showing
false coverage.

Package-test targets: `worker/pkg/keytool` (REG-01..09, WTAG-05), `coord/overlay`
(OVL-02, PLC-04), `coord/contact` (WINFO-05/06), `internal/billing` (PAY-06/09),
`internal/period` (RWD-08), `pkg/logship` (MON-04/05/06), `coord/gc/bloomfilter` (GC-02),
`internal/version/checker` + `versioncontrol/api` (UPD-02/03/04/08, already green).

Manual: UPD-10 (per-OS restart), RLY-08 (double NAT), MON-07/08 (Grafana), WTAG-08 and
AUD-08 (absence/parity assertions).

---

## Critical files

- `internal/testplanet/planet.go` - `Config`, `createPeers` ordering, peer lifecycle.
- `internal/testplanet/worker.go` - the `HavePublicAddress = true` forcing that the
  behind-NAT mode must bypass.
- `internal/testplanet/coord.go` - `cfg.Server.P2PAddress = ""` and
  `cfg.Overlay.DistinctNetwork = false`; where `Contact.RelayAddrs` gets set.
- `internal/testplanet/helpers.go` - reuse `AuditVerifier()`, `AuditSegmentForFile()`,
  `PieceHolders()`, `DropCoordPieces()`, `FindWorker()`; add `DepositWatcher()`.
- `internal/testplanet/reconfigure.go` - `Reconfigure.Coord` / `.Worker` hooks are how
  every gated subsystem gets enabled; `CombineCoord` / `CombineWorker` for stacking.
- `internal/testplanet/uplink.go` - `Upload`, `Download`, `DownloadRange`; add multipart
  and inline helpers for FS-02/03.
- `coord/peer.go` - gating flags `Audit.Enabled`, `Repair.Enabled`,
  `GC.BloomFilter.Enabled`, `GC.Sender.Enabled`, `Deposit.Enabled`; `Peer` struct is the
  test's handle on every subsystem.
- `coord/contact/service.go:56` - `RelayAddrs`, the coord side of relay advertisement.
- `coord/relay/peer.go` + `peer_test.go` - relay peer constructor and the proven AutoRelay
  worker-host wiring to copy.
- `coord/deposit/service.go` - `NewService(log, config, chain, store, rates)`, injectable
  as-is.
- `docs/test-plan.md` - updated in the same commit as each section's tests.
- `internal/testplanet/delete_contract_test.go` - the one existing e2e delete test; its
  fixture is upgraded from an inline segment to a real RS upload.
- `coord/file/service.go:298` / `:120` - `Delete` and `AuthorizeFileOwner`; the live-only
  contract lookup at `:135` is what TC-DEL-06 asserts.
- `coord/storage/service.go:116` - `DeleteContract`; `deleteBatchSize = 100` at `:30` is the
  TC-DEL-14 boundary; the resume path documented at `:113` is TC-DEL-17.
- `coord/db/file_repo.go:102,196` - the two delete CTEs (single-file, and batched
  per-contract).
- `coord/accounting/tally/tally.go` - `fillContractTallies` zero-seed, the TC-DEL-09
  invariant.

New files: `internal/testplanet/relay.go`, `versioncontrol.go`, `fakechain.go`,
`delete_file_test.go`, and one `<section>_test.go` per section.

## Verification

Prerequisites, both already required by the existing suite:

- Postgres: `export DEPIN_TEST_POSTGRES="postgresql://admin:admin123@localhost:5445/hub?sslmode=disable"`
  (or `-postgres-test-db`). Without it `testplanet.Run` skips, which reads as a pass -
  confirm the suite actually ran, do not trust a green with skips.
- `go-sdk`: `go.mod` pins `gitlab.internal/aioz-depin/go-sdk`. If a local `../go-sdk`
  replace is in play for edge/SDK work, the symlink must exist or `edgeserver` will not
  compile at all.

Per commit:

```
go build ./...
go test ./internal/testplanet/ -run '<the new tests>' -count=1 -v
```

Whole suite, at the end and before any merge:

```
go test ./internal/testplanet/ -count=1          # full planet suite
go test ./... -count=1                            # nothing else regressed
go test ./internal/testplanet/ -race -count=1     # planet runs many goroutine peers
```

Explicit checks beyond exit code:

- Count executed tests against the section table; a `SKIP` line means Postgres was not
  configured, not that the case passed.
- `TestEdgeserverMetadataHeaders` must be *fixed and failing before the fix* - it is
  currently asserting the opposite of what the handler does, so treat it as a bug fix with
  a red-then-green demonstration, not a refactor.
- For the relay commit, assert reservations exist (`WaitForReservations`) before asserting
  the data path, so a relay-bootstrap failure does not surface as a confusing upload error.
- Long-running keepalive cases (WINFO-10, RLY-04) hold connections idle past the keepalive
  interval; run them with an explicit `-timeout` and keep them out of the default fast
  path if they push the suite over budget.
- After the final commit, every row in `docs/test-plan.md` has a non-empty `Test` cell.
  Grep for empty cells as the completeness check.
