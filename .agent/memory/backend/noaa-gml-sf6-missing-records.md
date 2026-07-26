# noaa_gml/sf6 — missing-records incremental-fetch fix

Same bug class as [[ucdp-conflict-window-collapse]] and the usgs earthquake fix.

## Bug
`internal/datasource/noaa_gml/sf6`:
- `store.go` `LatestFetch` = `MAX(fetched_at)` (INGEST wall-clock).
- `fetcher.go` filter `if !latest.IsZero() && !date.After(latest) { continue }` compared each record's EVENT month against that ingest clock.
- Every published month is in the past relative to the last ingest, so after the first backfill EVERY record is dropped -> nothing new ever ingested. Monthly series = totally stalled.

## Fix (Template A: delete filter + add Upsert)
- Store had only plain `Create` (INSERT) + `UNIQUE INDEX idx_sf6_records_date` on `date` -> it did NOT dedup, so a pure filter-delete would hit dup-key errors on re-scan.
- Added `Store.Upsert` (ON CONFLICT (`date`) DO UPDATE value cols) mirroring marketstack/earthquake; `Service.Save` now calls `Upsert`.
- Extracted parse into pure `decodeRecords(io.Reader)` (behavior-preserving first), then deleted the watermark filter entirely. Full `sf6_mm_gl.txt` is re-scanned every poll; Upsert makes it idempotent and also absorbs NOAA revisions to past months.
- `LatestFetch`/`MAX(fetched_at)` KEPT — it is correct as the interval-skip gate ("did we fetch recently"); it was only wrong when misused as an event-date filter.

## Test
`fetcher_test.go` `Test_decodeRecords_keepsRecordsBelowFetchWatermark` — 3 monthly rows with event months before a later ingest watermark must all survive. RED (0/3) before fix, GREEN after.

## Note
Repo has a golines-style save-format hook (~80col) that reflows the WHOLE edited file (splits fn signatures / slog calls). Unavoidable; every edited datasource file gets it. Hook is file-scoped to the edited file only. Sibling packages (n2o, ucdp, usdm, awdb, earthquake, ...) were being fixed concurrently by other agents in the same worktree during this task — do not touch/revert them.
