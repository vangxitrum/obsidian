# OWM One Call Forecast Poller + API — Design

Date: 2026-07-13
Branch: feat/current-weather
Repo: /home/tuan/work/backend (aioz-map)
Status: Approved (brainstorming)

## Goal

Ingest weather forecast time-series data from the OpenWeatherMap One Call API for
every location in the `state_provinces` table, at multiple time steps over a
configurable range, and expose it via REST.

## Decisions (locked)

- Granularity: hourly + daily.
- Scope: ingestion + REST API.
- Range config: horizon + step.
- Endpoint version: config-driven (`base_url`). OWM has no public 4.0 One Call;
  the version string lives entirely in `base_url`, so 3.0 vs "4.0" is a config
  value, not a code assumption.
- Storage: upsert-latest by valid time (no revision history).
- Rate limiting: reuse `datafeed.DataFeed` poller path (consistent with open_meteo).

## Architecture

New datasource package `internal/datasource/open_weather_map/weather_forecast/`,
mirroring `open_meteo/weather_forecast`:
`config.go / model.go / dto.go / fetcher.go / service.go / store.go`.
REST wiring under `internal/api/weatherforecast/`.

Key efficiency point over the open_meteo hourly/daily split: One Call returns
`current` + `hourly` + `daily` in a single request. Therefore ONE fetcher / ONE
HTTP call per location per interval feeds both hourly and daily tables. Request
uses `exclude=current,minutely,alerts&units=metric`.

### Flow per tick

1. `store.ListLocations(ctx, maxLocations)` — stale-first ordering (locations with
   oldest `fetched_at` first), same as open_meteo.
2. For each location: One Call GET via `datafeed.DataFeed` (rate-limit + poll
   tracking).
3. Slice `hourly[]` / `daily[]` by config horizon + step → hourly + daily record
   sets.
4. Upsert both (batched).

Backfill goroutine on startup + ticker loop at `interval`, matching the open_meteo
`Run` pattern.

## Data model — two tables, upsert by valid time

Daily and hourly carry different fields, so two tables (matches the existing
open_meteo hourly/daily split).

### owm_hourly_forecast_records
`id, location_id (FK state_provinces), forecast_ts (UTC valid time),
temp, feels_like, pressure, humidity, dew_point, uvi, clouds, visibility,
wind_speed, wind_deg, wind_gust, pop, rain_1h, snow_1h,
weather_id, weather_main, weather_desc, weather_icon,
source, fetched_at, created_at, updated_at`

### owm_daily_forecast_records
`id, location_id (FK state_provinces), forecast_ts (UTC day),
sunrise, sunset, moonrise, moonset, moon_phase, summary,
temp_day, temp_min, temp_max, temp_night, temp_eve, temp_morn,
feels_like_day, feels_like_night, feels_like_eve, feels_like_morn,
pressure, humidity, dew_point, wind_speed, wind_deg, wind_gust,
clouds, pop, rain, snow, uvi,
weather_id, weather_main, weather_desc, weather_icon,
source, fetched_at, created_at, updated_at`

### Upsert semantics
Unique index on `(location_id, forecast_ts)` per table, `ON CONFLICT DO UPDATE`.
Each poll revises the forecast for a given valid-time slot instead of appending
duplicates; the table always holds the newest forecast per (location, time). No
forecast-revision history (that would be a separate append-only table if ever
needed).

## Config

New sub-config `sources.open_weather_map.weather_forecast`:

```
enabled          bool      # default false — dark until key/plan confirmed
api_key          string    # reuse OWM key
base_url         string    # e.g. https://api.openweathermap.org/data/3.0/onecall (version = config)
interval         duration  # poll cadence
hourly_horizon   int        # hours ahead, clamped <= 48
hourly_step      int        # keep every Nth hour (1 = all)
daily_horizon    int        # days ahead, clamped <= 8
timeout          duration
rps              float
burst            int
max_locations    int
```

Effective-getter helpers clamp `hourly_horizon<=48`, `daily_horizon<=8`,
`hourly_step>=1`, sensible defaults when zero.

## REST API

`GET /api/v1/weather/forecast?lat=&lng=`
- Resolves the nearest `state_province` via haversine (reuse the pattern in the
  open_meteo store `GetClosestByDateRange`).
- Returns `{ location, hourly: [...], daily: [...] }`.
- Optional params: `step=hourly|daily|both` (default both), `from` / `to` to
  window `forecast_ts`.
- Follows the existing `currentweather` endpoint + `response.GeneralResponse`
  conventions. Registered in the weather route group in `internal/api/router.go`.

## Registration

Wire in `internal/app/datasource.go` under the OWM group (`cfg.OpenWeatherMap` /
`GroupWeather`): construct store → service → fetcher, register the poller
alongside the existing OWM sources. API endpoint constructed in the API app
wiring (`internal/app/api.go` / `server.go`) like `currentweather`.

## Testing (e2e-leaning)

- Fetcher test (httptest): assert URL contract (`appid`, `units=metric`,
  `exclude=current,minutely,alerts`, lat/lon); canned One Call JSON → assert
  horizon/step slicing + field mapping for both hourly and daily.
- Store test: upsert dedupe — same `(location, forecast_ts)` inserted twice ⇒ one
  row, updated (not duplicated).
- Endpoint e2e: seed DB (location + forecast rows), hit handler, assert
  nearest-location resolution, response shape, `step` filter, and `from`/`to`
  windowing.

## Migration

`internal/database/migration/<ts>_owm_weather_forecast.{up,down}.sql` — create both
tables + unique indexes. Schema-qualify table names to avoid the known
golang-migrate search_path baseline leak.

## Out of scope

- Minutely precipitation and One Call alerts (excluded from the request).
- Forecast-revision history / verification scoring.
- Backfilling historical forecasts.
