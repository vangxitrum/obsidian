---
type: fact
tags: [aioz-map, backend, go, datasource, open_weather_map, air_pollution]
created: 2026-07-16
agent: subagent
---

# OWM current air-pollution persistence (MAP backend)

Extended `internal/datasource/open_weather_map/air_pollution/` (pkg
`air_pollution`) from on-demand-only to ALSO pre-fetch + persist current air
pollution per `state_provinces` row. Mirrors the [[owm-forecast-datasource]]
structural template.

## What was already there (kept intact)
- `service.go`: on-demand `Service` with `httpDoer *http.Client`,
  `NewService(log, cfg)`, `FetchByLatLng(ctx, lat, lng) (*AirPollutionResponse,
  error)` — wired to the live `/weather/air-pollution` endpoint
  (`internal/api/airpollution/endpoint.go`). Do NOT change its signature.
- `AirPollutionResponse` struct (coord + list[main.aqi + components + dt]) —
  `decodeRecords` REUSES this shape instead of duplicating.
- `config.go`: had Enabled/APIKey/BaseURL/RPS.

## What was added (files under the pkg only)
- **model.go**: `AirPollutionRecord` -> table `owm_air_pollution_records`. FK
  `StateProvinceID *uint` + `Location types.StateProvince`. Unique index
  `idx_owm_ap_sp_time` on (state_province_id, measured_at). `MeasuredAt` from
  OWM `dt` (unix->UTC). `AQI int`, 8 float64 components (`PM25` col `pm2_5`).
- **dto.go**: `AirPollutionData` intermediate + `dataToRecord` + `AirPollutionDTO`
  (JSON serving shape) + `ToDTO(*AirPollutionRecord) *AirPollutionDTO` (nil-safe)
  for the forecast endpoint to embed.
- **fetcher.go**: PURE `decodeRecords(io.Reader) ([]AirPollutionData, error)`
  (no high-water/event-time filter — avoids the clock-mismatch bug class).
  `Fetcher` + `NewFetcher(log, cfg, feed *datafeed.DataFeed, service)`,
  `Run(ctx)` (initial fetch + ticker on Interval, default 1h), `Name()`
  = "open_weather_map.air_pollution". Iterates ListLocations(ctx,100), 1
  `/air_pollution?lat=&lon=&appid=` call/loc via `feed.Do`, sets StateProvinceID
  + lat/lon, service.Save.
- **store.go**: `NewStore(log, db)`, `Upsert` (OnConflict (state_province_id,
  measured_at) DO UPDATE), `ListLocations(ctx,limit)` (oldest-MAX(fetched_at)
  first, mirror forecast), `GetLatestByLocation(ctx, spID uint)
  (*AirPollutionRecord, error)` (most-recent measured_at), `GetLatestByCoords(
  ctx, lat, lng float64) (*AirPollutionRecord, error)` (haversine-nearest loc's
  latest). Both nil-return when absent.
- **service.go** (extended): added `store *Store` field +
  `NewStoreService(log, store)` (poller/serving; httpDoer nil) +
  `Save(ctx, []AirPollutionData, fetchedAt)` + `GetLatestByLocation` /
  `GetLatestByCoords` thin wrappers over store. `NewService(log, cfg)` unchanged.
- **config.go** (extended): added `Interval time.Duration`, `Timeout
  time.Duration`, `Burst int` (kept Enabled/APIKey/BaseURL/RPS). Additive — the
  parent's `air_pollution.Config` mapstructure `air_pollution` still binds.

## Migration (added, allowed — not the pkg dir)
`internal/database/migration/20260716000001_owm_air_pollution.{up,down}.sql`
creates `owm_air_pollution_records` + unique index `idx_owm_ap_sp_time`.
Timestamp is one tick after the forecast migration (20260716000000). Column `no`
is a non-reserved Postgres keyword so it's fine unquoted. No central prod
AutoMigrate in this repo (SQL migrations only) — same as forecast.

## Parent-side wiring still TODO (NOT done — shared files untouched)
- config.go already has `AirPollution air_pollution.Config` mapstructure
  `air_pollution` + defaults (enabled/base_url/rps). Poller needs Interval/Timeout/
  Burst defaults added (currently only rps=0.5). The on-demand config key is
  shared, so poller reads the SAME `air_pollution.*` block.
- app/datasource.go: add a poller block gated `cfg.AirPollution.Enabled &&
  inGroup(group, GroupWeather)` — copy the OWMForecast block (lines ~293-344)
  verbatim, swap "forecast"->"air_pollution", `owmforecast`->`air_pollution`,
  and use `air_pollution.NewStoreService(log, air_pollution.NewStore(log, db))`
  (NOT NewService — that's the on-demand HTTP one). datasource.Load/New with NO
  auth (fetcher adds appid itself). Two lifecycle.Items (Close datasource, Run).
- Forecast merge: the OWM forecast endpoint can embed AP via
  `apStore.GetLatestByLocation(ctx, spID)` / `GetLatestByCoords(ctx, lat, lng)`
  -> `air_pollution.ToDTO(rec)` (nil-safe pointer, embed as omitempty). Construct
  ONE shared `air_pollution.NewStore(log, db)` and thread it into the forecast
  endpoint's service.

Verified: RED (`undefined: decodeRecords`) -> GREEN (`Test_decodeRecords_
parsesAirPollution` PASS). go test/vet/build clean for the pkg; app+api build
clean with the additive changes. Related: [[owm-forecast-datasource]]
[[datasource-incremental-fetch-clock-mismatch]] [[map-overview]]
