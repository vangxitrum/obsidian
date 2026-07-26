---
type: fact
tags: [aioz-map, backend, go, datasource, open_weather_map, forecast]
created: 2026-07-16
agent: subagent
---

# OWM 5-day/3-hour forecast datasource package (MAP backend)

New self-contained package `internal/datasource/open_weather_map/forecast/`
(pkg `forecast`). Pre-fetches OWM `/data/2.5/forecast` (40 x 3-hourly steps) per
`state_provinces` row and upserts.

## Layout (mirrors open_meteo/weather_forecast structural template)
config.go, model.go, store.go, service.go, fetcher.go, dto.go, endpoint.go,
fetcher_test.go. No mcp.go (skipped as heavy).

## Key design
- Record: `ForecastRecord` -> table `owm_forecast_records`. FK
  `StateProvinceID *uint` + `Location types.StateProvince`. Unique index
  `idx_owm_forecast_sp_time` on (state_province_id, forecast_time) so re-fetch
  upserts a step in place. `ForecastTime` from OWM `dt` (unix->UTC).
- `decodeRecords(io.Reader) ([]ForecastData, error)` is the PURE TDD unit: one
  record per `list` entry, NO high-water-mark/event-time filter (avoids the
  incremental-fetch clock-mismatch bug class). City coord applied per step; the
  fetcher overrides StateProvinceID + lat/lon per location.
- Store `Upsert` uses `clause.OnConflict` on the two-col index DO UPDATE value
  cols. `ListLocations(ctx, limit)` = oldest-`MAX(fetched_at)`-first (same query
  shape as open_meteo, adapted to owm_forecast_records). `ListByLocation` /
  `GetByCoords` (haversine nearest) for serving.
- Fetcher `Run`: `go runFetch` initial, then ticker on `config.Interval`
  (defaults 3h if <=0). `runFetch` iterates `ListLocations(ctx, 100)`, one
  `/forecast?lat=&lon=&units=metric&appid=` call per loc via
  `feed.Do(ctx, req, fn)`, upsert per location.
- Source = `constants.OpenWeatherMapDataProviderName` ("open_weather_map").

## Constructors (match open_meteo arities for wiring)
`NewStore(log, db)`, `NewService(log, store)`,
`NewFetcher(log, cfg Config, feed *datafeed.DataFeed, service)` + alias
`NewCurrentFetcher(...)` same sig, `NewEndpoint(log, service)`.
`Config{Enabled,APIKey,BaseURL,Interval,Timeout,RPS,Burst}` (mapstructure tags).

## Wiring the parent still must do (NOT done here)
- AutoMigrate: register `forecast.ForecastRecord` in `internal/database/*`.
- Construct in `internal/app/*`: NewStore->NewService->NewFetcher (needs a
  datafeed.DataFeed for OWM with RPS/Burst/Timeout) + register fetcher.Run.
- Router: add `GET /weather/owm-forecast` -> `Endpoint.Get`.
- Config: add `open_weather_map.forecast` block + config.example.yaml.

Verified: go test (RED `undefined: decodeRecords` -> GREEN), go vet, go build all
clean for the package. gofmt clean.

## Parent-side wiring — DONE (2026-07-16)
Shared-file wiring completed; `go build ./...`, `go vet`, forecast pkg test all pass.
- **Config naming decision**: field `OWMForecast owmforecast.Config` mapstructure
  `open_weather_map_forecast` (flat top-level key, like `current_weather`/`air_pollution`
  which are also flat sub-service keys). Import aliased `owmforecast` in every shared file.
- **Table creation**: repo has NO central/production GORM AutoMigrate (only per-package
  *test* AutoMigrate + golang-migrate SQL files). Post-baseline datasources (sun_times,
  worldtides) create tables via SQL migration pairs. So I added
  `internal/database/migration/20260716000000_owm_forecast.{up,down}.sql` creating
  `owm_forecast_records` + unique index `idx_owm_forecast_sp_time (state_province_id,
  forecast_time)`. Did NOT touch `internal/app/datasource_test.go` AutoMigrate list (none
  of the weather siblings are in it — wrong analog).
- **Poller**: `internal/app/datasource.go` new block after OWM weather-map block, gated
  `cfg.OWMForecast.Enabled && inGroup(group, GroupWeather)`. Builds a fresh
  `datasource.Load/New("open_weather_map", ...)` with NO auth (fetcher adds `appid`
  itself — do NOT reuse the current_weather DS which has WithAuthQuery appid → double
  appid), `datafeed.New(log,"open_weather_map.forecast",...)`, `owmforecast.NewFetcher`,
  two lifecycle.Items (Close `open_weather_map.forecast.datasource`, Run
  `open_weather_map.forecast`).
- **Endpoint**: api.go constructs `owmForecastEndpoint` → threaded through
  `api.NewServer` (server.go) → `registerRoutes` (router.go). Route
  `GET /api/v1/weather/owm-forecast` → `owmForecastEndpoint.Get`.
- **Gotcha**: goimports save-hook strips an aliased import if added in a separate edit
  before its usage exists, and auto-adds the alias once usage exists (caused a duplicate
  import in router.go). Add usage first, or expect goimports to insert the aliased import
  for you.
