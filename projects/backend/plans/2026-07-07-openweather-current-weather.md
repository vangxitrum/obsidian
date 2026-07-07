---
type: plan
project: backend
tags: [backend, openweathermap, api, weather]
created: 2026-07-07
status: implemented
---

# OpenWeather Current Weather 2.5 + support matrix (2026-07-07)

## Context
aioz-map backend progressively exposes OpenWeatherMap data. Before this change only Air Pollution JSON was decoded; weather maps were tiles-only. Goal: implement everything the FREE OWM tier offers that we did not already have.

## Key finding (OWM tiering, verified vs openweathermap.org/price)
- Current Weather 2.5 = FREE -> implemented.
- Weather Maps 1.0 = FREE but only 5 tile layers (clouds/precip/pressure/wind/temp) - already had these.
- Weather Maps 2.0 (dew point TD2, snow depth SD0, etc.) = PAID (Professional+). Cannot be served with the free/v1 key. So no NEW free map layer exists.
- Excluded per request (already served by the separate weathertiles GRIB2 datasource): UV index, apparent temperature, visibility, cloud ceiling.

## Delivered
1. New free endpoint GET /api/v1/weather/current-weather?lat=&lng= (units=metric). Returns full OWM blob: temp, feels_like, humidity, pressure, visibility, wind speed/dir/gust, cloud cover %, precip mm, snow mm, weather code/description.
   - New pkg internal/datasource/open_weather_map/current_weather (config.go, service.go) cloned from air_pollution.
   - New pkg internal/api/currentweather (dto.go, endpoint.go) cloned from airpollution.
   - Config: current_weather block; key via env AIOZ_MAP_CURRENT_WEATHER_API_KEY. Wired through config.go, app/api.go, api/server.go, api/router.go.
2. Support-matrix README at internal/datasource/open_weather_map/README.md (status per field: implemented / already-in-weathertiles / paid / not-in-OWM-use-Open-Meteo).
3. Maps 2.0 (v2) registry: DEFERRED - confirmed paywalled, dormant until a paid key is bought.

## Verification
go build + go vet clean. Unit tests (httptest mock OWM) pass: URL contract (/weather, units=metric, appid), full-blob decode, upstream-401 surfaced, endpoint 400 on missing params, 200 on valid.

## Not from OWM (use Open-Meteo, already integrated): cloud base, freezing rain/ice, hail, soil temp/moisture, pressure levels (1000-10 hPa), CAPE.
