# Plan: raise depin test coverage to 80% of product code

## Context

Coverage was measured for the first time on 2026-08-09 (full suite, 102 packages,
0 failures, `-coverpkg=./...` so the testplanet e2e suite gets credited against
the packages it actually exercises). There was no prior number.

| Scope | Coverage |
| --- | --- |
| Raw, everything | 41.2% (24,556/59,587) |
| Excluding generated `pkg/pb` | 59.1% (17,645/29,849) |
| **Product code** (also excl. `cmd/*` mains, `dev/*` tooling) | **62.78% (17,393/27,704)** |

41.2% cross-checks exactly against `go tool cover -func`.

Target agreed with the user: **80% of product code**, i.e. generated protobuf,
`cmd/*` CLI wiring and `dev/*` tooling are outside the denominator. That is a
deliberate choice: those 2,145 statements are mostly flag plumbing where tests
catch little, and including them would add ~30% more work for no bug-finding.

The gap is concrete:

- Product baseline: **62.78%**, uncovered pool 10,311 statements
- After deleting confirmed dead code (Phase A): **64.01%** (17,393/27,172)
- 80% of 27,172 = 21,737 covered, so **+4,344 statements must be newly covered**
- Remaining uncovered pool after Phase A is 9,779, so this means covering **44%
  of everything still untested**

Phases B-E below identify roughly 5,030 statements of realistically reachable
coverage, leaving ~16% margin over the 4,344 needed.

Third requirement, added by the user: **the suite is already too slow in CI.**
This is sequenced first, not last. `internal/testplanet` alone takes **525s
locally without `-race`**, and CI runs `go test -tags purego -race -vet=off
-timeout 30m ./...` as a single unsharded job (`.gitlab/ci/test.yml:47`), where
`-race` typically costs another 2-10x. Adding 4,344 statements of new tests to a
suite already close to its 30-minute timeout would simply break CI, so Phase 0.5
must land before Phases A-F.

Second requirement, added by the user: **worker versions roll out slowly, so the
fleet is always running a mix of versions, and tests must cover from the minimum
supported version through the latest.** Phase F covers this. It is worth being
explicit that Phase F contributes only ~120 statements toward the 4,344 - it is
not where the percentage comes from. It is where the production risk is, because
a mixed-version fleet is the normal state and nothing tests it today.

## Non-negotiable: coverage must not be gamed

This is the main risk. A statement counts as covered if *any* test binary
executed it, including incidentally during setup. It is trivial to move the
number without testing anything, and this session already caught several false
greens (a repair test that passed while repair did nothing; five OVL rows marked
covered from filenames alone).

Rules for every phase:

1. **Every test asserts observable behaviour.** No test whose only effect is to
   call a function and check `err == nil`.
2. **Measure per-package too, not just the global `-coverpkg` number.** Run the
   target package with plain `go test -cover ./pkg/foo/` - that only counts the
   package's *own* tests and cannot be inflated by incidental execution
   elsewhere. Both numbers should move together; if only the global one moves,
   the tests are not real.
3. **Error paths are the point.** The uncovered 37% is overwhelmingly error
   handling, which is exactly where untested code hides bugs. Prefer one test
   that forces a failure over three that repeat the happy path.
4. **Do not test generated or vendored code** to inflate the ratio.
5. Every new bug found gets a row in `docs/test-plan.md`, matching the
   convention already in place (150+ rows, each naming its test).

## Phase 0: make the metric reproducible and gate it

Nothing else matters if the number silently regresses.

- Add a `coverage` target to the Makefile with the exact command and the
  exclusion list baked in, so everyone measures the same thing:
  ```
  go test ./... -coverpkg=./... -coverprofile=cover.out -count=1
  # then filter: drop /pkg/pb/, ^aioz-depin/cmd/, ^aioz-depin/dev/
  ```
  Requires `DEPIN_TEST_POSTGRES`; without it testplanet skips and the number
  silently collapses. The target must fail loudly if the variable is unset.
- Add a **ratchet check in CI**: fail if product coverage drops below the
  recorded baseline. Ratchet upward as phases land. A one-off push to 80% that
  is not gated will decay.
- **Fix the Go toolchain properly.** `/usr/local/go` is 1.23.2 while `go.mod`
  needs 1.25, so `GOTOOLCHAIN=auto` downloads a toolchain module whose tool dir
  ships only 7 prebuilt tools, missing `covdata` - every `-coverpkg` run dies
  with `go: no such tool "covdata"`. Plain `-cover` is unaffected, which is why
  this stays hidden. It was worked around by building `covdata` from the
  toolchain source into the module cache; that lives in `~/go/pkg/mod` and any
  `go clean -modcache` erases it. Install a real Go 1.25 instead, otherwise CI
  cannot reproduce the measurement.

## Phase 0.5: cut CI runtime (blocks everything else)

Measured facts:

- `internal/testplanet`: **71 tests, 525s, zero `t.Parallel()`** - strictly
  sequential, ~7.4s each.
- Each planet calls `db.Migrate()` (`internal/testplanet/coord.go:137`), and
  there are **43 migration files**. That is **3,053 migration executions** per
  suite run, each its own transaction plus a version-table write.
- CI is one job running everything with `-race` and a 30m timeout, no sharding.
- No fixed sleeps in the testplanet source; the polling that does exist is
  `waitForOnlineWorkers` at 200ms granularity against a 30s deadline.

**Step 1 is to measure the split** between migration cost and peer-boot cost for
a single planet. Everything below is ordered by expected impact, but the order
should be confirmed against that measurement rather than assumed.

1. **Migrate once, clone per planet.** Run the 43 migrations a single time into
   a template schema at package setup, then give each planet a schema created
   from that template (Postgres `CREATE DATABASE ... TEMPLATE`, or capture the
   post-migration DDL once and replay it as one exec per planet). Turns 3,053
   migration executions into 43. Touches `internal/testplanet/coord.go:137` and
   `coord/db/coorddbtest/schema.go`.
2. **Parallelise testplanet.** Add `t.Parallel()` and run with `-parallel N`.
   Verify first that planets are actually independent: per-planet dynamic ports,
   per-coord Postgres schemas (already the case), and no shared global state
   (monkit registry is the main suspect). The connection budget already fits -
   each coord is capped at 8 open + 4 idle, and CI runs Postgres with
   `max_connections=500`, so roughly 30 concurrent planets are affordable. Even
   `-parallel 4` should cut testplanet wall time close to 4x.
3. **Shard the CI job.** Split fast unit packages from `internal/testplanet` so
   unit feedback does not wait on e2e, then use GitLab `parallel: N` with `-run`
   partitioning to spread testplanet across runners. This is independent of
   step 2 and composes with it.
4. **Re-scope `-race`** (decided: MRs fast, nightly full race). Drop `-race`
   from the MR `test` job and add a scheduled nightly job on `main` that runs the
   full suite with `-race`. This is the single biggest multiplier. The accepted
   tradeoff is that a data race can merge and is caught within a day instead of
   at MR time, so the nightly job must be loud on failure, not just red in a
   dashboard nobody opens.
5. **Right-size planets.** Several tests boot `WorkerCount: 6` where 4
   (== `TotalShares`) is sufficient; each extra worker is a full peer with its
   own databases. Audit and reduce where fleet size is not the subject.
6. **Replace polling with triggers** in `waitForOnlineWorkers` - 200ms
   granularity per test adds up across 71 tests.

Target: testplanet under ~2 minutes wall time in CI, and the total `test` job
comfortably inside its timeout with room for Phases A-F to add to it.

## Phase A: delete dead code (-532 statements, +1.23pp, no tests written)

Eight packages have **zero external references** (verified by grep for their
import path across all `*.go`, including `_test.go`). They are 0% covered and
cost nothing to remove:

| Package | Stmts |
| --- | --- |
| `pkg/common/ranger` | 222 |
| `pkg/eestream/scheduler` | 71 |
| `pkg/workerclient` | 66 |
| `pkg/common/sync2/combiner` | 53 |
| `internal/http_response` | 43 |
| `pkg/common/paths` | 33 |
| `pkg/common/context2` | 23 |
| `worker/pkg/p2p` | 21 |

`pkg/common/ranger` is a byte-for-byte-sized duplicate of `pkg/ranger` (222
statements each); `pkg/ranger` is the live one with 4 importers. Confirm each
deletion with `go build ./... && go test ./...` before moving on.

## Phase B: pure-logic packages (~+1,750)

No infrastructure needed, so these are the cheapest statements in the repo and
should be table-driven. Several are correctness-critical.

| Package | Uncov | Now | Target |
| --- | --- | --- | --- |
| `pkg/common/sync2` | 422 | 25.0% | 90% |
| `pkg/ranger` | 222 | **0.0%** | 90% |
| `pkg/encryption` | 219 | **10.6%** | 90% |
| `internal/mud` | 234 | 53.9% | 85% |
| `internal/vo` | 215 | 74.5% | 92% |
| `pkg/common` | 176 | 5.4% | 90% |
| `pkg/common/pkcrypto` | 153 | **0.0%** | 90% |
| `internal/grpcutil` | 145 | 24.5% | 90% |
| `internal/lrucache` | 112 | 37.8% | 90% |
| `pkg/common/time2` | 104 | 25.2% | 90% |

Two of these deserve attention beyond the number:

- **`pkg/ranger` is 0% covered yet imported by `pkg/encryption/transform.go`,
  `pkg/encryption/pad.go`, `pkg/eestream/decode.go` and `encode.go`** - the core
  download path, and ranged downloads shipped recently. Zero statements execute
  across the entire suite including e2e. Either the constructors are never
  reached at runtime (in which case part of it is also dead) or the ranged read
  path is genuinely untested. Investigate before writing tests.
- **`pkg/common/pkcrypto` is 0% with 51 importers** - the most widely depended-on
  untested package in the repo, and it is crypto.

## Phase C: identity and process (~+550)

| Package | Uncov | Now | Target |
| --- | --- | --- | --- |
| `pkg/identity` | 304 | 38.3% | 85% |
| `pkg/process` | 314 | 31.4% | 80% |
| `certificate/authorization` | 93 | 0.0% | 85% |

`pkg/identity` and `certificate/authorization` back node identity and the
signing service (`cmd/keytool` drives them). Security-relevant and currently
thin. `pkg/process` is config/logging wiring; note `pkg/process/logging.go`
already has a fixed bug in it (the `pretty` encoder dropping `log.With` fields),
which is exactly the kind of regression a test here would have caught.

## Phase D: worker storage layer (~+1,400)

The largest single concentration of untested product code.

| Package | Uncov | Now | Target |
| --- | --- | --- | --- |
| `worker/pieces` | 765 | **20.5%** | 80% |
| `worker/pkg/blobstore/filestore` | 373 | 53.0% | 85% |
| `worker/pkg/hashstore` | 275 | 87.3% | 93% |
| `worker/pkg/filewalker` (+ `lazyfilewalker`) | 200 | ~32% | 80% |
| `worker/db` | 138 | 45.0% | 80% |
| `worker/monitor` | 113 | 24.2% | 80% |

These are filesystem-backed and test with `t.TempDir()`, no cluster needed.
`worker/monitor` is where the **`SharedDisk.DiskSpace` bug already documented in
`docs/test-plan.md`** lives (`storageStatus` declared and never assigned, so
reported Total/Free are always 0 and `MinimumDiskSpace` enforces nothing) - fix
it under test as part of this phase.

## Phase E: coord and worker service paths (~+1,330)

These already sit at 65-77%; the remainder is error handling. Extend the
existing testplanet suite rather than writing new harnesses - `internal/testplanet`
already provides coord + workers + uplinks + relay, and `Reconfigure` hooks make
failure injection straightforward (the disk-capacity tests just used exactly
this pattern).

| Package | Uncov | Now |
| --- | --- | --- |
| `coord/db` | 261 | 77.1% |
| `coord/file` | 226 | 75.2% |
| `worker/piecestore` | 207 | 69.1% |
| `coord` (peer wiring) | 177 | 50.6% |
| `coord/segmentverify` | 175 | 71.0% |
| `coord/audit` | 152 | 74.5% |
| `coord/placement` | 141 | 54.1% |
| `coord/repair/repairer` | 135 | 73.5% |
| `pkg/eestream` | 131 | 72.1% |
| `coord/relay` | 117 | 65.7% |
| `worker/orders` + `ordersfile` | 212 | ~62% |
| `coord/server` | 106 | 44.8% |
| `worker/retain`, `worker/contact` | 185 | ~68% |

Known open defects that belong here, already recorded in `docs/test-plan.md`:
the burned order serial that makes per-piece retry inert and masks the real
error, and the coordinator never filtering selection on capacity.

## Phase F: mixed-version fleet, minimum supported version to latest (~+120)

Because rollout is slow, coord always talks to workers spanning several
versions. Today nothing tests that. What exists:

- `pkg/version.ShouldUpdateVersion` (`pkg/version/version.go:251`) is the gate:
  below-minimum forces an update to `Minimum`, otherwise the rollout cursor
  decides. **63% covered**, and two pre-existing bugs were already found in this
  package during the worker-updater work - it is fragile and undertested.
- `versioncontrol` serves per-binary `minimum` / `suggested` / `rollout`.
- coord **records** a worker's version (`coord/contact/version.go`,
  `carryForwardVersion`) but **never gates on it**, and selection has no version
  filter (`coord/overlay`, `coord/placement`). Old workers keep serving.
- testplanet does not set worker versions at all, so every planet is uniform.

### F0: declare the real minimum supported version

`dev/versioncontrol/config.yaml` sets `minimum: v0.0.0`, which means "never
force an update" - there is effectively no declared floor.

Decided: **the floor is the oldest worker build still running on any fleet.**
First task is therefore to determine what that actually is - the fleets are
reported to be on v0.0.5/v0.0.7 while git tags stop at `v0.0.2`, so the deployed
builds are not all reachable from a tag. Establish the real answer by querying
the coordinator for the distinct worker versions it has recorded (coord already
persists a per-worker `version` via `carryForwardVersion`), take the minimum,
and write that into the versioncontrol config as the declared floor.

If the oldest deployed build has no corresponding tag, tag it retroactively from
the commit it was built at, otherwise F3 has nothing to build or diff against.

### F1: version-decision matrix (pure, cheap) - `pkg/version` 63% -> 90%

Table-driven tests over `ShouldUpdateVersion`, one row per branch:

| current vs config | expected |
| --- | --- |
| below minimum | returns `Minimum`, "below minimum allowed" |
| at minimum, below suggested, rollout candidate | returns `Suggested` |
| at minimum, below suggested, not candidate | empty, no update |
| at suggested | "up to date" |
| above suggested (dev build, rollback) | "up to date", never a downgrade |
| malformed semver in minimum/suggested | error returned, no panic |

Plus `isRolloutCandidate` at cursor 0 / 100 / partial, asserting the same node ID
lands on the same side of the cursor across calls (an unstable seed would make
the fleet flap). Also covers `pkg/version/buildinfo` (14 stmts, currently 0%).

### F2: mixed-version fleet e2e (testplanet)

Add a per-worker version knob to testplanet (a `Version` field set through the
existing `Reconfigure.Worker` hook, feeding `worker/contact` `self.Version`), then
assert against a fleet spanning the declared minimum through HEAD:

- Upload and download succeed with a mixed-version fleet.
- Coord records each worker's own distinct version, not the last one to check in.
- **Selection does not filter by version.** Pin this explicitly. If someone later
  adds version gating, this test fails loudly rather than silently stranding the
  slow-rolling tail of the fleet.
- Repair, audit and GC all operate across a mixed fleet.
- A worker reporting an **empty** version is still selected and served. This was
  a real bug (an older build omitted `Version` from `WorkerInfo` and sent empty);
  the fix must not regress into treating empty as "too old".

### F3: wire compatibility - the actual risk

Nothing prevents a proto change from breaking an older worker mid-rollout.

- **Add buf with a breaking-change check in CI.** There is no `buf.yaml` in the
  repo today. `buf breaking --against <minimum-version-tag>` catches field
  renumbering, removals and type changes before merge. This is the single
  highest-value item in the version story: it is cheap, runs on every PR, and
  covers most of what a full version matrix would.
- **One cross-version smoke test**, not an N x M matrix. Build the worker at the
  declared minimum version (from its git tag, or reuse the existing Docker worker
  package from `fleet.sh package`) and run it as an external process against the
  current coord: check-in, upload, download. A full matrix costs a lot and the
  breaking-change check already covers the common failure.

## Critical files

- `Makefile` - new `coverage` target (Phase 0)
- `.gitlab/ci/test.yml:47` - the single unsharded `-race` job (Phase 0.5)
- `internal/testplanet/coord.go:137` + `coord/db/coorddbtest/schema.go` -
  per-planet migration, the main Phase 0.5 target
- CI config - ratchet gate (Phase 0), buf breaking-change check (F3)
- `docs/test-plan.md` - new rows per phase, existing 150-row + findings format
- `internal/testplanet/` - Phase E extends this; `Reconfigure` for fault
  injection; F2 adds the per-worker version knob
- `pkg/version/version.go:251` - `ShouldUpdateVersion`, the F1 subject
- `coord/contact/version.go`, `coord/overlay`, `coord/placement` - the
  no-version-gating contract F2 pins
- `dev/versioncontrol/config.yaml` - where the declared minimum lives (F0)
- `buf.yaml` - new (F3)
- Deleted in Phase A: the eight package directories listed above

## Verification

Per phase:

```
go build ./...
go test ./<touched packages>/ -count=1            # must pass
go test ./<touched package>/ -cover -count=1      # per-package number moved
go test ./internal/testplanet/ -count=1           # e2e still green (~525s)
make coverage                                     # global product number moved
```

Checks beyond exit code:

- **Both** the per-package and global numbers move. Global-only means the tests
  are not really exercising the package.
- `DEPIN_TEST_POSTGRES` must be set or testplanet skips silently and the whole
  measurement is wrong - count executed tests, never trust a green with skips.
- Known pre-existing flake: `TestGetExpired` in `worker/pieces` (time-sensitive,
  fails only in a full `./worker/...` run; verified pre-existing). Phase D
  touches this package and should fix it.
- Phase A: `go build ./... && go test ./...` after each deletion.

For Phase F specifically:

```
go test ./pkg/version/... -cover -count=1                      # F1: 63% -> 90%
go test ./internal/testplanet/ -run 'Version' -count=1 -v      # F2 mixed fleet
buf breaking --against '.git#tag=<minimum-version-tag>'        # F3
```

F2 must be checked for false greens the same way the disk-capacity tests were:
assert a *specific* worker holds a piece, not just that the upload succeeded,
otherwise a mixed-version planet proves nothing that a uniform one does not.

For Phase 0.5, the check is wall time plus unchanged results:

```
go test ./internal/testplanet/ -count=1                 # baseline today: 525s
go test ./internal/testplanet/ -count=1 -parallel 8     # after parallelisation
go test ./internal/testplanet/ -count=1 -race           # still passes, still green
```

Same test count passing before and after. A speedup that comes from tests
silently skipping is the failure mode to watch: compare `--- PASS` counts, not
just the total time and exit code.

## Exit criteria

- `internal/testplanet` under ~2 minutes in CI; the `test` job well inside its
  timeout with headroom for the new tests.
- Product-code coverage >= 80%, with the CI ratchet set to that number.
- Every new finding recorded as a row in `docs/test-plan.md`.
- A minimum supported worker version is declared (F0), the version-decision
  matrix is covered (F1), a mixed-version fleet is exercised end to end (F2),
  and proto breaking changes are blocked in CI against that minimum (F3).
