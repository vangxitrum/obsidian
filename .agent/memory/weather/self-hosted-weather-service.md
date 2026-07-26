---
type: decision
tags: [weather, architecture, grib2, open-meteo, tiles, python, s3, rabbitmq]
created: 2026-07-17
agent: main
---

Greenfield project at `/home/tuan/work/map/weather` (empty repo as of 2026-07-17): a
self-hosted weather service with two features — (1) point data for a lat/lng covering every
variable Open-Meteo supports (current + forecast), and (2) GRIB2 -> XYZ map tiles at
configurable zoom, with click-on-map to read the exact value behind the pixel.

**Confirmed decisions (user):**
- Point data: **self-host Open-Meteo** (`ghcr.io/open-meteo/open-meteo` container) + thin proxy.
- Language: **Python** (FastAPI; shells out to GDAL; rasterio for point sampling).
- Click-to-value: **sample the source raster** (raw GRIB2 in S3) for the exact model value.
- Infra: **S3 + RabbitMQ** event queue (mirror parser-service).

**Why:** Self-hosting Open-Meteo gives every variable free/unlimited and replaces paid OWM
point calls; the tile pipeline replaces OWM tiles; click-to-value is net-new (neither existing
service has it).

**How to apply / port from siblings (NOT this repo):**
- GRIB2->tile GDAL pipeline: `aioz-map/parser-service/internal/processor/grib2/pipeline.go`
  (gdal_translate -> gdalwarp EPSG:3857 -> lanczos upsample -> blur_tif.py -> gdaldem
  color-relief -> gdal2tiles.py --xyz -z). Color tables in `parser-service/colors/`.
- NOMADS GFS download + cycle detect: `parser-service/cmd/download-raw-sample/main.go`
  (`buildGFSURL`, `detectGFSRun`).
- Layer/legend/units: `backend/internal/datasource/weathertiles/layers.go`.
- Tile S3 upload/serve: `parser-service/internal/storage/tiles.go`, `.../internal/api/server.go`.
- MapLibre viewer to port + extend with click: `parser-service/frontend/index.html`.
- The "tile matrix" README (Open-Meteo serves no tiles): `backend/internal/datasource/open_weather_map/README.md`.

**Status (2026-07-17):** scaffold built (Python/FastAPI, docker-compose) and VERIFIED end-to-end via
`docker compose` on a live GFS cycle. Point proxy returns live Open-Meteo JSON. FULL tile pipeline
runs automatically from a clean bucket: scheduler (NOMADS->S3 + publish) -> RabbitMQ -> worker
(GDAL render -> 816 tile objects in MinIO + completed event) -> api (`/tiles` 302 presign to a valid
PNG; `/api/layers/{k}/value` samples raw GRIB via gdallocationinfo). render/blur/grib is a faithful
port of parser-service (verbatim GDAL flags). Hardened the broker startup race with `app/mq.py`
retry-connect (worker+scheduler). GOTCHA: `docker compose build <svc>` builds only that service's
image; worker/scheduler have their own `weather-worker`/`weather-scheduler` images - run
`docker compose build` (no arg) to rebuild all, else they run stale code. NOT yet run: open-meteo
container model sync (heavy).
NOTE: `/api/point` is a passthrough proxy, so callers use Open-Meteo's param names
(`latitude`/`longitude`), NOT lat/lon. Click-to-value endpoint uses lat/lon.

TILE AESTHETICS GOTCHA: blur `sigma` is applied AFTER the Nx upsample, so a small sigma smooths
almost nothing of the real field -> speckle/noise. For upsample 8, sigma ~16 actually smooths the
data. Color tables (`colors/*.txt`) should fade to transparent (alpha 0) where conditions are
"good/clear" so it works as an overlay (don't paint an opaque background), and use a smooth
severity ramp (pale green -> yellow -> orange -> red -> magenta), not hard neon aviation bands.
Viewer default basemap is dark CARTO so warm hazard blobs pop. To preview tile styling, render in a
one-off `weather-api` container and Read the PNGs (agent can view images).

LAYERS + SWAGGER (added 2026-07-17): 8 GFS tile layers - cloud-ceiling, visibility, temperature-2m,
relative-humidity-2m, total-cloud-cover, precipitation-rate, mslp, wind-gust. Adding a layer = one
config block (grib_var/grib_level NOMADS filter names + color table + legend) + a `colors/*.txt`.
Unit conversion via `value_scale`/`value_offset` in the layer config, applied in `value.py` (raw GRIB
-> display: Pa*0.01=hPa, kg/m2/s*3600=mm/h). GOTCHA: GDAL's GRIB driver already decodes GFS
temperature to CELSIUS (`GRIB_UNIT=[C]`, ~-74..43), NOT Kelvin - so no offset, and the color table
must be in °C. Swagger/OpenAPI is FastAPI built-in at `/docs` (+/openapi.json); enriched with
response models (`app/schemas.py`), documented query params, and tag groups. NOMADS level filter
names that work: 2_m_above_ground, surface, mean_sea_level, entire_atmosphere.

BROWSER-DEMO SETUP: `scratchpad/dc.browser.yml` override publishes api :8000 + minio :9100, sets
`S3_PUBLIC_ENDPOINT=http://localhost:9100` (tile presign must be signed for a browser-reachable host
- SigV4 binds the host; the api's own S3 reads still use the internal `minio:9000`), and bind-mounts
`./frontend` so viewer HTML edits show on refresh without a rebuild.

MODEL SYNC RUN + VERIFIED (2026-07-17): `open-meteo-sync` + `open-meteo-api` now started and
self-hosted point data returns REAL values end-to-end (direct serve + our `/api/point` proxy):
pressure_msl 1012.2 hPa, visibility 24140 m, cape 600, cin 18 at Berlin. **CRITICAL ARCHITECTURE
CORRECTION:** the original assumption "self-host Open-Meteo = every variable free/unlimited" is
WRONG for the FREE tier. Open-Meteo's free AWS open-data bucket (`https://openmeteo.s3.amazonaws.com/data/<model>/`)
hosts only a SUBSET of ncep_gfs025. **NOT in free bucket** (KeyCount 0): temperature_2m,
precipitation, relative_humidity_2m, wind_speed_10m, wind_u/v_component_10m, cloud_cover,
surface_temperature, snowfall - i.e. exactly the popular surface vars we want. **IN free bucket**
(has chunks): pressure_msl, cape, convective_inhibition, visibility, freezing_level_height, and the
pressure-level fields (cloud_cover_1000hPa..50hPa, geopotential_height_*, etc.). Getting the common
surface vars requires the PAID apidata.open-meteo.com S3 sync subscription (api key / S3 creds).
Chunk layout: `data/<model>/<var>/chunk_<N>.om`; current chunk ~1030 (chunk_time_length 481h,
hourly). Sync of 5 free vars x 2 chunks (--past-days 1) = ~1.1 GB, ~2 min at ~10 MB/s. Sync of the
UNAVAILABLE vars silently downloads only `static/HSURF.om` + meta.json (399 KB) then "Repeat in 60
min" -> serve returns all-null (grid structure present, no values). GOTCHAS: (1) host port 8080
taken by `sish-manager`, 8081 also taken -> `open-meteo-api` published on host **8082**:8080;
internal proxy still uses `open-meteo-api:8080` so unaffected. (2) A one-off `docker run` against the
`./data/open-meteo` volume WITHOUT `--user 0` hits "Permission denied" because compose sync
(`user:"0"`) created the dirs root-owned; use the compose service or add `--user 0`. compose
`open-meteo-sync` command edited to the available-vars set (see docker-compose.yml comment).
DECISION NEEDED: (a) accept subset for free point data, (b) buy apidata S3 sync for full surface
vars, or (c) keep proxying PUBLIC api.open-meteo.com for full free coverage (rate-limited) and
self-host only tiles. IMPORTANT: this bucket limitation does NOT affect tiles - the tile pipeline
pulls surface vars (TMP, PRATE, RH, TCDC, GUST...) directly from NOMADS GFS GRIB, which is fully free.

MERGED POINT ENDPOINTS (built 2026-07-17, v0.3.0, VERIFIED e2e): added `/api/current` and
`/api/forecast` that return "everything we have at a point" without a per-variable list.
`/api/current` merges self-hosted Open-Meteo current vars (the synced free subset, nulls dropped)
+ ALL 8 GFS tile-layer values sampled from the stored f000 analysis GRIB (temp/precip/RH/cloud/mslp/
gust/ceiling/visibility, each with legend band + cycle). `/api/forecast` = Open-Meteo hourly series
(forecast_days 1-16); GFS is analysis-only (f000) so it is intentionally ABSENT from forecast.
IMPLEMENTATION: `app/merged.py` (new router, tag "merged point data"); refactored `app/value.py`
to extract reusable `sample_layer(layer, lat, lon, cycle=None) -> dict|None` (shared by
click-to-value route + /api/current; returns None on no-grib/no-data instead of raising);
`open_meteo.variables` list added to config.example.yaml (single source of truth for synced vars,
keep in sync with the open-meteo-sync command); schemas CurrentResponse/ForecastResponse/MergedValue
in `app/schemas.py`; registered in main.py BEFORE the static "/" mount. Swagger description/tags
updated (v0.3.0) so it no longer overpromises "every variable" - now points users to the merged
endpoints + notes surface vars come from GFS tiles. E2E tests: `tests/test_merged.py` (3 tests, hit
live :8000, skip if down) - all pass. GFS point value != Open-Meteo point value for overlapping vars
(e.g. pressure) because different model/cycle - expected, kept under separate `open_meteo`/
`gfs_analysis` namespaces (no flat merge, no collision on `visibility`).

GFS MULTI-HOUR FORECAST SERIES (built 2026-07-17, VERIFIED e2e, 4/4 tests pass): `/api/forecast`
now returns a `gfs` block alongside `open_meteo` - a value series for ALL 8 GFS layers at the
forecast-hour cadence, filling the gap that the free Open-Meteo bucket can't (temp/precip/wind/etc.
now have a real forecast, not just f000). DESIGN: pre-stack at ingest. Scheduler downloads every
forecast hour per layer (concurrent, `nomads.download_concurrency`), stacks them into ONE multi-band
GeoTIFF (band k = forecast_hours[k]) via `pipeline/stack.py` (gdalbuildvrt -separate -> gdal_translate),
uploads `stack/{prod}/{domain}/{cycle}/{layer}.tif` + a `.json` sidecar (init_time, actual
forecast_hours, unit, scale). At request time `app/value.py:sample_stack()` does ONE gdallocationinfo
(no -b) -> all bands = whole series in one call (8 calls/request, not 8xN). `app/merged.py:_gfs_forecast()`
aligns layers to a shared time axis (dict time->value, union sorted) and clips to forecast_days.
Cadence config: `nomads.forecast_hours_range {start,stop,step}` (default 0..240 step 3 = 81 steps,
10-day; helper `config.forecast_hours()`, explicit `forecast_hours:` list overrides). Tiles + /api/current
UNCHANGED (still f000; scheduler still uploads f000 raw grib + publishes tile events for f000 only;
other hours live only inside the stack - no 600+ loose objects).

CRITICAL GRIB GOTCHA (band selection): some GFS vars ship TWO messages per forecast hour>0 - PRATE
and TCDC return band1 = instantaneous (GRIB PDS template PDTN=0, GRIB_FORECAST_SECONDS = valid time)
and band2 = interval-average (PDTN=8, has a time-range in the template); f000 has only band1. Naive
stacking gave 161 bands (1 + 80x2) != 81 hours, so sample_stack dropped those 2 layers. FIX:
`gdalbuildvrt -separate -b 1` takes band 1 (instantaneous) of every input -> uniform 81 bands,
consistent with the f000 tile / click-to-value band-1 convention. Diagnose band mismatch with
`gdalinfo <stack.tif> | grep -c 'Band '` vs sidecar forecast_hours length; identify instantaneous
band via `GRIB_PDS_PDTN=0`. GOTCHA 2 (dev only): api caches stacks in `/tmp/weather-grib` keyed by
S3 key; re-ingesting the SAME cycle overwrites same-key stacks but the api serves the STALE cached
copy until the cache is cleared (`rm -rf /tmp/weather-grib/*`) or api restarts. Harmless in prod
(new cycle = new key), only bit us when re-ingesting one cycle after the band fix. Stack footprint:
~219 MB/layer (81-band global 0.25) = ~1.75 GB/cycle; NO old-cycle pruning yet (scheduler dedup TODO).

OPS GOTCHA (docker, 2026-07-17): host port **9000 is PERMANENTLY held** by an unrelated container's
docker-proxy (started Jul-15, container-ip 172.27.0.4) - that's why minio publishes on **9100**, not
9000. Base docker-compose.yml maps minio 9000:9000, which ALWAYS conflicts. The browser override
(session scratchpad `dc.browser.yml`) now pins minio with `ports: !override [ "9100:9000" ]` +
api `S3_PUBLIC_ENDPOINT=http://localhost:9100` + frontend bind-mount. CRITICAL: when recreating api,
use `docker compose ... up -d --no-deps --force-recreate api` - WITHOUT `--no-deps`, compose pulls in
the minio dependency, tries to recreate it on base port 9000, fails, and leaves minio DOWN (Created,
not Up), taking S3 with it. Recreate command that works:
`docker compose -f docker-compose.yml -f <override> up -d --no-deps --force-recreate minio api`.
open-meteo-api host port also bumped 8080->**8082** (8080/8081 taken); internal proxy still
open-meteo-api:8080. Rebuild after api-side code change: `docker compose build api` (api is its own
image), then the recreate command above.

Plan doc: `/home/tuan/.claude/plans/now-open-meteo-gives-you-pure-sonnet.md`. Vault mirror to
`Projects/weather/plans/` PENDING - hermes router (`kr/claude-sonnet-4.5-agentic`) returns HTTP 402
MONTHLY_REQUEST_COUNT; retry when quota resets. See [[hermes-obsidian-writes]].
