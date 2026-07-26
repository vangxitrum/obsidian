---
type: fact
tags: [aioz-map, backend, go, datasource, bug-class, incremental-fetch]
created: 2026-07-14
agent: main
---

# Datasource incremental-fetch clock-mismatch bug class (MAP backend)

A systemic "missing records" bug found across **12 datasource fetchers** in
`/home/tuan/work/backend/internal/datasource/`. Same root cause everywhere.

## Root cause
Fetcher derives its high-water mark from `MAX(fetched_at)` (the **ingest
wall-clock** — when rows were last written) then either (a) compares it against
each record's **event/source date** to skip "old" records, or (b) uses it as an
event-date query bound (`starttime`/`beginDate`/`startdate`/ArcGIS `time`/`event_date >=`).
`fetched_at` and event-date are **different clocks**. Upstreams publish/revise with
lag, so any record whose event date ≤ last ingest time but which appears later is
dropped forever. For slow (daily/monthly) series it means **nothing new is ingested
after the first backfill**. UCDP variant: month-window collapses (`start≈now > end=now-2mo`).

## Two fix templates
- **Template A** — pure in-memory filter → delete it, rely on store `Upsert`
  (`ON CONFLICT` on a stable unique key) to dedup a re-scanned window. Reference:
  `usgs_earthquake/earthquake` (`decodeRecords`, no filter) [[earthquake-missing-records]].
- **Template B** — event-date query bound → drive it from `MAX(<event_date column>)`
  (add a store method), overlap back by a safe revision-lag, keep window bounded,
  rely on upsert to dedup the overlap. Reference (correct pre-existing): `marketstack/price`.

**Decision rule:** verify the store write path first. Plain `Create`/INSERT + unique
index → CANNOT bare-delete filter (dup-key). Either add Upsert (Template A) or use
event-date watermark (Template B).

## Outbox gotcha (important)
Feeds wiring a **per-record alert outbox** (`alertService.CreateOutbox`, wired in
`internal/app/datasource.go`) must NOT re-save already-ingested rows each poll:
`AlertOutbox` has `uniqueIndex(alert_type, source_id)` and `CreateOutbox` logs an
**error** on every duplicate → log spam + failed inserts per row per poll. So a
Template-A "re-save whole file every poll" is only safe when the feed has **no
outbox**. co2 & ch4 have the outbox → they use an event-date **gate**
(`filterNewByDate(recs, MAX(date))`) so only genuinely-new records reach Save.
n2o & sf6 have no outbox → plain Template A is fine.
Trade-off of the gate: NOAA **revisions** to already-stored months are not
re-ingested (accepted to avoid outbox spam). Alternative if revisions matter:
upsert-all + gate only the outbox call to new records.

## Per-package outcome (all: reproduce→failing test→fix→GREEN, go vet clean)
- `usgs_earthquake/earthquake` — A (delete filter; upsert on event_id).
- `noaa_gml/co2` — B/gate (MAX(date) `filterNewByDate`, plain Create; has outbox).
- `noaa_gml/ch4` — A+gate (added Upsert on date; has outbox → added `filterNewByDate` gate in a follow-up main-thread pass).
- `noaa_gml/n2o`, `noaa_gml/sf6` — A (added Upsert on date; no outbox).
- `nsidc_sii/daily_sii` — B (per-region MAX(date); store has no unique key so gate does dedup; also `LatestFetch` MAX→Order().Take() for sqlite test portability).
- `ucdp/conflict` — window-collapse: added `LatestEventDate=MAX(start_date)`, clamp `start≤end` so window can't collapse; upsert on conflict_id.
- `relief_web/conflict_news` — delete filter (upsert on article_id), watermark MAX(published_at), window −7d, bounded pagination; also guarded a Source[0] panic.
- `awdb/snow_cover` — B (MAX(date) drives beginDate; plain insert no unique key → event-date watermark).
- `nasa_coolr/landslide` — B (MAX(date) −30d, clamp to 2024-01-01; upsert on event_id) + added ArcGIS resultOffset pagination.
- `usdm/drought` — B (MAX(date) −14d; upsert on (county_fips,date)).
- `arcgis/chokepoints` — B (MAX(date) −7d, end=now+24h; upsert on object_id) + added pagination.

## Open follow-ups (NOT done)
- earthquake initial full backfill from 2026-04-01 has no `minmagnitude`/`limit` →
  can hit USGS 20k-event cap → HTTP 400 hard-fail. [[earthquake-missing-records]]
- `ucdp/conflict` `fetchByVersion` does `page++; continue` on HTTP error with no
  break → infinite-loop hazard if a not-yet-published version month is requested.
- Pre-existing `go vet` errors (unrelated, untouched pkgs): `nws/severe_weather`,
  `open_meteo/weather_forecast_daily`, `usni/usni_news`, `coinglass/whaletransfer`.
- Repo save-format hook is golines-style (~80col) but committed code isn't →
  inflates every touched-file diff. Output is still gofmt-clean.

Related: [[chokepoints-fetched-at-watermark]] [[ucdp-conflict-window-collapse]]
[[noaa-gml-co2-missing-records]] [[noaa-gml-ch4-missing-records]] [[noaa-gml-sf6-missing-records]] [[map-overview]]
