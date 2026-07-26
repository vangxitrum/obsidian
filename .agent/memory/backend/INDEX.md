# backend (aioz-map) — memory index

Project: `/home/tuan/work/backend` — aioz-map backend Go service.

- [[map-overview]] — what MAP/aioz-map backend is: Go datasource poller + Gin REST API over Postgres.
- [[datasource-incremental-fetch-clock-mismatch]] — SYSTEMIC "missing records" bug in 12 fetchers: MAX(fetched_at) ingest-clock used as/against event-date. Two fix templates (A delete-filter+upsert / B event-date watermark) + outbox gotcha. Read first for any datasource incremental-fetch work.
- [[chokepoints-fetched-at-watermark]] — ArcGIS chokepoints incremental-fetch bug: MAX(fetched_at) used as event-time `time` extent + no pagination. Fixed with MAX(date) watermark + overlap + resultOffset paging.
- [[ucdp-conflict-window-collapse]] — incremental-fetch window-collapse bug (MAX(fetched_at) watermark) fixed in `internal/datasource/ucdp/conflict`.
- [[noaa-gml-sf6-missing-records]] — same bug class in `internal/datasource/noaa_gml/sf6`; fixed via delete-filter + add Upsert (Template A).
- [[noaa-gml-co2-missing-records]] — same bug class in `internal/datasource/noaa_gml/co2`; fixed via event-date watermark MAX(date) + plain Create (Template B), NOT Upsert — because co2 has an active per-record outbox (unlike sf6) that Template A would spam.
- [[noaa-gml-ch4-missing-records]] — same bug class in `internal/datasource/noaa_gml/ch4`; fixed via delete-filter + add Upsert, THEN (main-thread follow-up) added event-date `filterNewByDate` gate + `LatestDate` because ch4 has an outbox (like co2) that re-save-all would spam. See [[datasource-incremental-fetch-clock-mismatch]].

- [[owm-forecast-datasource]] — new self-contained `internal/datasource/open_weather_map/forecast` package (OWM 5-day/3-hour forecast pre-fetch into `owm_forecast_records`, upsert on (state_province_id, forecast_time)); mirrors open_meteo/weather_forecast. Includes parent-side wiring checklist.
- [[owm-air-pollution-datasource]] — extended `internal/datasource/open_weather_map/air_pollution` to ALSO pre-fetch+persist current air pollution per state_province (`owm_air_pollution_records`, upsert on (state_province_id, measured_at)); kept on-demand `FetchByLatLng`. Adds `GetLatestByLocation`/`GetLatestByCoords`+`ToDTO` for merging into the forecast endpoint. Includes parent wiring checklist.
- [[owm-ondemand-request-reduction]] — Stage 3: cut live OWM calls on the two on-demand endpoints, no response-shape change, pollers untouched. Air-pollution endpoint now serves nearest-city from `owm_air_pollution_records` (`Store.GetLatestByCoords` + new `RecordToResponse` mapper), live fallback on miss. Current-weather (no pre-fetch table) gets a 30m / 5000-cap `ExpiringLRUOf` keyed on `%.1f,%.1f` coords. Endpoint constructor `airpollution.NewEndpoint` gained a `*air_pollution.Store` arg (api.go wiring).

Design spec + docs moved out of the repo into the Obsidian vault: `/home/tuan/personal/tuan/projects/backend/{specs,docs}/` (index: `projects/backend/INDEX.md`). Repo `docs/` now holds only generated swagger.
