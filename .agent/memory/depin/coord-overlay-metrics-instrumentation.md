---
type: fact
tags: [depin, coord-overlay, monkit, metrics]
created: 2026-07-10
agent: subagent (task-8)
---

Task 8 (commit 81dca60, branch `feat/coord-metrics-collection`, same series as
[[coord-accounting-metrics-instrumentation]]): monkit gauges + refresh-duration for
coord/overlay's two selection caches, at the real refresh-completion site in each
`Refresh()` (right after the new state is swapped in under the lock, before the
existing `c.log.Debug`).

`UploadSelectionCache.Refresh` (coord/overlay/uploadselection.go) already fetches and
logs a reputable/new split via `contact.Store.SelectParticipatingWorkersSplit` — reused
those exact categories: `upload_cache_reputable_workers`, `upload_cache_new_workers`,
`upload_cache_eligible_workers` (= reputable+new, the pool's own doc comment already
calls this "eligible"), `upload_cache_refresh_duration`.

`DownloadSelectionCache.Refresh` (coord/overlay/downloadselection.go) originally got
**only** `download_cache_eligible_workers` + `download_cache_refresh_duration` — the
implementer reasoned `SelectDownloadableWorkers`'s flat pool (deliberately including
suspended workers, unlike upload) had no reputable/new categorization to reuse.
**Review found that reasoning wrong** (fix commit 7f9b67e, same branch, same day):
`contact.SelectedWorker.Vetted` is populated on every worker `SelectDownloadableWorkers`
returns, because `WorkerRepository.SelectDownloadableWorkers`
(coord/db/worker_repo.go) calls the same `selectWorkerRows`→`projectWorker` path as
`SelectParticipatingWorkersSplit` — the exact source the upload cache's split already
reads from. Suspension-inclusion and Vetted/reputable-new are orthogonal axes; fixing
the gauges didn't touch suspension logic. Added `download_cache_reputable_workers` +
`download_cache_new_workers` by tallying `Vetted`/`!Vetted` in the existing
`byAlias`-building loop (no extra pass); `download_cache_eligible_workers` is now
their sum. All 4 gauges + duration now mirror the upload cache exactly. Test fixture
`aliasWorker` (downloadselection_test.go) left untouched; added a sibling
`vettedAliasWorker(alias int64, vetted bool)` for the new gauge assertions. Report:
`.superpowers/sdd/task-8-report.md` "## Fix Report" section.

coord/overlay had no package-level `mon`; added coord/overlay/monkit.go (mirrors
coord/placement/monkit.go). All metrics recorded only on refresh success, not on a
failed source call (mirrors [[coord-accounting-metrics-instrumentation]]'s
tally.Service.Tally pattern). Tests use the by-now-standard withTestMon
scope-swap pattern (see [[coord-audit-metrics-instrumentation]]) against the existing
fakeSplitReader/fakeDownloadReader fixtures already in uploadselection_test.go /
downloadselection_test.go, plus new erroringSplitReader/erroringDownloadReader fakes
for the error-skips-gauges tests.
