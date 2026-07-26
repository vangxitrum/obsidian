# UCDP conflict fetcher — window-collapse incremental-fetch bug

Package: `internal/datasource/ucdp/conflict`. Fixed 2026-07-14 (branch `fix/earthquake-missing-record`).

## Symptom
Fetcher ingested nothing after the initial backfill. UCDP is a monthly-versioned
dataset ("YY.0.M"); the fetcher loops month-versions from a `start` watermark up
to `end`.

## Root cause (WINDOW COLLAPSE variant of the fetched_at-watermark bug class)
- `runBackfill` used `LatestFetch` = `MAX(fetched_at)` (ingest wall-clock ≈ now)
  as the window `start`, and `fetch` computed `end = now.AddDate(0,-2,0)`
  (2-month UCDP publication lag).
- In steady state `start ≈ now` is ~2 months AFTER `end`, so the loop
  `for !cur.After(end)` never ran → empty result → nothing ingested.
- Same root class as the earthquake fetcher fix: watermark from `MAX(fetched_at)`
  (ingest time) used as an event-time bound.

## Fix
- Added `Store.LatestEventDate` = `MAX(start_date)` (DATA watermark), mirroring
  marketstack/price's `MAX(date)`. `runBackfill` now uses it for the fetch
  window, and keeps `MAX(fetched_at)` ONLY as the wall-clock re-poll skip guard.
- Extracted pure helper `computeVersionWindows(latest, now) []string` (versions
  to fetch). It rounds start/end to month, and CLAMPS `start` to `end` so the
  window can never collapse and always (re)fetches at least the newest published
  version.
- Safe to re-scan overlap: `Store.Upsert` does `ON CONFLICT (conflict_id) DO
  UPDATE`, so months are deduped.

## Notes
- Kept the 2-month publication lag as `end` (NOT `now`): `fetchByVersion` loops
  `page++; continue` on HTTP error with no break, so querying a not-yet-published
  version month would infinite-loop. Pre-existing hazard, left as-is (out of scope).
- Repo save-format hook (golines/gofumpt) reflows the WHOLE file on any edit
  (splits long func signatures, strips blank lines). `git checkout` + re-apply
  does NOT avoid it — the hook re-applies deterministically. All sibling fetcher
  fixes show the same noise. Accept it.
- Tests: `fetcher_test.go::TestComputeVersionWindows_*` (table-free, pure).
