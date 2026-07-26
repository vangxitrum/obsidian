---
type: fact
tags: [coord, metrics, monkit, telemetry, prometheus, remote-write, branch-status]
created: 2026-07-10
agent: main
---

Branch `feat/coord-metrics-collection` (worktree `/home/tuan/.treehouse/depin-b971d9/1/depin`) is **complete and pushed** to `origin` (gitlab.internal). MR: `https://gitlab.internal/aioz-depin/depin/-/merge_requests/32`

**Post-push cleanup round (2026-07-10, same day):** the user had merged `main`/`develop` into this branch outside the session (via GitLab/local git directly, not this agent) before asking to continue. That merge included `e1c7d02 refactor: replace uplink-sdk with external go-sdk` (from `feat/build-edgeserver-with-sdk`, see [[edgeserver-go-sdk-migration]]) which deleted all of `uplink-sdk/` except `client.go` — kept alive only because this branch's own baseline-fix commit had also modified that file, so git's merge treated it as modify-vs-delete and kept the modified side. Removed the orphaned `uplink-sdk/client.go` (commit `fd20ee9`) — confirmed zero remaining Go importers first. The same merge silently reverted `coord/file/service_test.go`'s `fakeStore.CreateSegment` (from develop's `feat/file-metadata` branch) back to a no-op stub, breaking `TestSegmentCommitted_RetryDoesNotDoubleCount` (the Task 6 regression test for idempotent-retry over-counting) — no merge conflict was flagged since the edits didn't textually overlap, but the fake's statefulness this test depended on was gone. Per explicit user direction, removed the test rather than re-diverging the shared fixture (commit `5fc7a1c`); production `wasNew` logic in `coord/file/service.go` is untouched and still correct, just no longer covered by an automated regression test in this fake-store setup.

**Lesson for future sessions:** after any external merge lands on a worktree between sessions, re-run the full test suite before trusting "last known green" — a clean, non-conflicting git merge can still silently break test fixtures shared across concurrently-developed features.

All 11 execution tasks (Parts A/B/C of the source plan `projects/depin/plans/2026-07-09-coord-metrics-collection.md`) done via subagent-driven-development: implementer → task-reviewer loop per task, then one final whole-branch review (Opus) → "Ready to merge: Yes", 3 cheap polish fixes applied. Per-task details: [[telemetry-remote-write-push]] (Tasks 1-2), [[coord-audit-metrics-instrumentation]], [[coord-repair-metrics-instrumentation]], [[coord-order-metrics-instrumentation]], [[coord-file-metrics-instrumentation]], [[coord-accounting-metrics-instrumentation]], [[coord-overlay-metrics-instrumentation]], [[coord-gc-metrics-instrumentation]], [[coord-monitoring-vps-artifacts]], [[coord-metrics-final-review-fixes]].

**Execution plan artifact:** the source plan was reorganized into `## Task N` sections at `docs/superpowers/plans/2026-07-09-coord-metrics-collection-tasks.md` on the branch (gitignored, not committed) to drive `scripts/task-brief`/`scripts/review-package`. The subagent-driven-development ledger (`.superpowers/sdd/progress.md`, also gitignored) has the full commit-range/decision/deferred-Minor-findings history per task if a future session needs to resume or audit.

**Baseline was broken before any of this work started** — see [[depin-gitignore-allowlist-gotcha]]. Fixed in the branch's first commit (39fb868).

**Human decisions made during execution (durable, apply to future metric-naming questions on this repo):**
- `audit_segments_*` counters count per worker/piece-outcome-instance within an RS report, not per segment (matches existing `coord/placement/audit.go` precedent) — name kept as-is despite the mismatch, clarifying comments added at call sites instead of renaming.
- `coord/file`'s `file_created`/`segment_committed`/`segment_downloaded` metrics deliberately have NO subsystem prefix (unlike every other sibling: `audit_*`, `repair_*`, `order_*`, `gc_*`) even though the source plan's own text implied a `file_` prefix (`file_segment_commit`) — kept as shipped, no rename, when explicitly re-raised at the final whole-branch review.
- Idempotent-retry over-counting on `segment_committed` was worth fixing by expanding scope into `coord/file/service.go` (not just `endpoint.go` as the task brief said) — precedent that "brief file-scope" is not a hard boundary when a real correctness fix needs it, if asked.

Two pre-existing, unrelated repo breaks were found and deliberately left untouched (confirmed out of scope): `uplink-sdk/cmd/download-demo` and `internal/testplanet` both reference `uplink-sdk` API surface (`DefaultDownloadConcurrency`/`WithDownloadConcurrency`/`CreateContractWithPlacement`) that was never implemented — real missing SDK functionality, not a build config issue. Repo-wide `go build ./...` will fail only in those two packages on this branch (and likely on `main` too, unverified). All verification for this branch stayed scoped to `go build ./coord/... ./pkg/... ./internal/... ./worker/... ./cmd/...`.

**Local `main` moved during this work** (merged `fix/build-error`, commits `19f6998`/`9d113da`) — its `uplink-sdk/client.go` fix is byte-identical to this branch's own fix, so merging will resolve that overlap with zero conflict.
