# Incremental-fetch watermark bug (class) — backend datasources

## The bug
Datasource fetchers compute the incremental query's `beginDate`/`startdate`
(an EVENT/OBSERVATION-date bound) from `MAX(fetched_at)` — the INGEST wall-clock.
Since fetched_at advances with wall time but source data is published with lag
and later revised, the lower bound races past observation dates that were not
yet published, so those records are never (re-)fetched → "missing records".

## The fix (target pattern = marketstack/price)
Drive the window from `MAX(<real event-date column>)` (event-date watermark),
via a store method (e.g. `LatestObservationDate` = `MAX(date)`). Keep
`MAX(fetched_at)` ONLY for the poll-interval throttle (legit wall-clock use).
If the store PLAIN-INSERTS (no upsert on a stable unique key), the event-date
watermark is the safe route (overlap-back needs upsert to dedup).

## snow_cover (fixed 2026-07-14, branch fix/earthquake-missing-record)
- Bug: `fetcher.go` fetch() `beginDate = latest.AddDate(0,0,1)` where latest =
  `store.LatestFetch()` = `MAX(fetched_at)`, used as AWDB `beginDate` query bound.
- Store `Create()` is a PLAIN gorm insert (no OnConflict); model has no
  (station,date) unique key → overlap would duplicate. So chose event-date route.
- Added `Store.LatestObservationDate()` = `MAX(date)`; extracted pure helper
  `backfillBeginDate(latestObs, latestFetch, defaultStart)`; runBackfill now uses
  fetched_at only for the throttle and obs-date watermark for the window.
- Test: `TestBackfillBeginDate_UsesObservationWatermarkNotFetchedAt`.
- Sibling `awdb/snow_station` is a one-time backfill — do NOT touch.
- NOTE: repo save hook is `golines` (short line width) but committed code is NOT
  golines-formatted → editing via Edit/Write reflows whole files (huge noise
  diff). Apply changes via Bash/python + `gofmt -w` to keep diffs minimal.

## Same class also being fixed in parallel: usdm/drought, noaa_gml/{n2o,sf6},
## ucdp/conflict, usgs_earthquake, arcgis/chokepoints, nasa_coolr/landslide.
