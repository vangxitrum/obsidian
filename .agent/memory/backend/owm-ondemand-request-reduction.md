# OWM on-demand request reduction (Stage 3)

Cut live OpenWeatherMap calls on the two on-demand endpoints, WITHOUT changing
any API response shape and WITHOUT touching the scheduled pollers.

## Air pollution (`/api/v1/weather/air-pollution`)
- Serve from the poller-warmed `owm_air_pollution_records` table (nearest city
  via `Store.GetLatestByCoords`), falling back to a live OWM call only on a cold
  miss (nil record) or a store error.
- Endpoint `internal/api/airpollution/endpoint.go` now takes `*air_pollution.Store`
  in addition to the live `*air_pollution.Service`. Wiring in
  `internal/app/api.go` (~L621): construct `air_pollution.NewStore(log, db)` and
  pass it as the 3rd arg to `airpollution.NewEndpoint(log, svc, store)`.
- Mapper `air_pollution.RecordToResponse(rec) *AirPollutionResponse` lives in
  `internal/datasource/open_weather_map/air_pollution/dto.go` (next to `ToDTO`).
  It rebuilds coord{lat,lon} + a one-element list{dt=MeasuredAt.Unix(),
  main.aqi, components{8}}. To construct the list element cleanly, the anonymous
  `List []struct{...}` element in `service.go` was factored into a named
  `AirPollutionListItem` (JSON tags unchanged -> byte-identical shape; verified
  by marshaling: `{"coord":{"lon","lat"},"list":[{"main":{"aqi"},"components":{co,no,no2,o3,so2,pm2_5,pm10,nh3},"dt"}]}`).

## Current weather (`/api/v1/weather/current-weather`)
- No pre-fetch table exists, so added an `ExpiringLRUOf[*CurrentWeatherResponse]`
  (`pkg/lrucache`) to the `current_weather.Service`. Consts in `service.go`:
  `cacheTTL = 30*time.Minute`, `cacheCapacity = 5000`. Cache name "current_weather".
- Key = `cacheKey(lat,lng) = fmt.Sprintf("%.1f,%.1f", lat, lng)` (~0.1deg /
  nearest-city cell) so nearby requests share one OWM call per TTL window.
- Refactor: live HTTP body moved to private `fetchLive`; `FetchByLatLng` now does
  GetCached -> fetchLive -> Add. Same pattern as `solar_panel_output/fetcher.go`.

## Tests (all pass)
- `air_pollution/dto_test.go`: `TestRecordToResponse_ReconstructsLiveShape`,
  `TestRecordToResponse_NilRecord`.
- `current_weather/service_test.go`: `TestCacheKey_QuantizesToNearestCity`
  (careful: pick coords NOT straddling a .05 rounding boundary),
  `TestFetchByLatLng_CacheServesNearbyWithoutSecondCall` (httptest counter, 2
  nearby calls -> 1 upstream hit).

No migrations, no go.mod, no poller/forecast changes. `go build ./...`, `go vet`,
package tests all clean.
