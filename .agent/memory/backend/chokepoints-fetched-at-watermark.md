# ArcGIS chokepoints fetcher — fetched_at-watermark incremental-fetch bug

Package: `internal/datasource/arcgis/chokepoints`. Fixed 2026-07-14 (branch `fix/earthquake-missing-record`).

## Symptom
Daily chokepoint records go missing after initial backfill; late-published /
revised features silently dropped, and large result sets truncated.

## Root cause (same class as earthquake / ucdp fixes)
- `fetcher.go` `fetch`: `start := latest` where `latest = MAX(fetched_at)`
  (`store.go LatestFetch`), passed to the ArcGIS `time` extent
  `q.Set("time", start.UnixMilli()..now)`. The `time` extent filters the layer's
  event `Date` field, so using the ingest wall-clock (which races ahead of the
  event dates, e.g. fetched_at 14:30 > today's midnight Date) drops features.
- Query had `where=1=1`, `orderByFields=Date DESC`, NO `resultRecordCount` /
  `resultOffset` / `exceededTransferLimit` handling → silent truncation at the
  FeatureServer maxRecordCount.

## Fix
- Added `Store.LatestEventDate` = `MAX(date)` (event-date watermark), mirroring
  marketstack/price `MAX(date)`. `runBackfill` drives the window from it and
  keeps `MAX(fetched_at)` ONLY as the wall-clock re-poll skip guard.
- Extracted pure helper `timeExtent(watermark, now) (start, end)`:
  `start = watermark - 7d` overlap lag (or `now - 30d` if no data),
  `end = now + 24h` timezone padding. Overlap is safe because `Store.Upsert`
  does `ON CONFLICT (object_id) DO UPDATE` (object_id is `gorm:"uniqueIndex"`,
  a stable unique key) → re-fetched rows deduped.
- Added bounded pagination in `fetch`: loop `resultOffset` += page size,
  `resultRecordCount=1000`, continue while `ExceededTransferLimit`, capped at
  100 pages. Extracted `mapFeatures` for the feature→ChokepointData mapping.

## Notes
- Store write-path was ALREADY a proper upsert on `object_id`, so the
  overlap-and-dedup strategy is safe (verified, not assumed).
- Test: `fetcher_test.go::Test_timeExtent_includesLatePublishedFeaturesBeforeWatermark`
  (pure, no DB). RED with `start = watermark` (no overlap), GREEN after overlap.
- Save-format hook reflows store.go on every edit (strips blank lines, wraps the
  `List` signature); `git checkout` + re-apply does NOT avoid it. Same noise on
  all sibling fetcher fixes. Accept it — only `LatestEventDate` is the real change.
