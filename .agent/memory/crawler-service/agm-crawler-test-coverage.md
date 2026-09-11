---
title: crawler-service test coverage, seams, and two defects found
type: project
updated: 2026-08-21
---

# crawler-service - test seams and findings (2026-08-21)

Added ingest-side tests for the AGM BE Self-Review tickets. See
[[agm-selfreview-test-coverage]] for the full 113-TC matrix.

## How to test a fetcher here

Two mechanics, both already established in the repo:

1. Fake the package's **private `feed` interface**, embedding `datafeed.Feed` so
   unmodelled calls panic instead of silently passing. Model:
   `internal/open_weather_map/air_pollution/fetcher_test.go`.
   NOTE: several packages' private `feed` is WIDER than `datafeed.Feed`
   (adds `FetchBody`, `StoreFileAndReference*`), so embedding alone is not enough.
2. Build `&Fetcher{...}` **by struct literal** - the `feed` field is unexported and
   `NewFetcher` only takes a concrete `*datafeed.DataFeed`. Tests must be in-package.
   Model: `internal/earthquake/shakemap/fetcher_test.go`.

11 of 14 fetchers are testable offline via `Config{BaseURL: srv.URL}`.
Where `fetch()` writes to a relative `data/` dir, use `t.Chdir(t.TempDir())`.

## Two defects found

- **Nil deref in `internal/fire/gwis_nasa`** (FIXED). Retry loop logged
  `err.Error()` unconditionally, but `FetchBody` can return `(nil, nil)` for an
  empty-but-successful response. Fetchers run as **bare goroutines in
  `internal/app/app.go` with no recover**, so this panicked the WHOLE crawler
  process. Pinned by `TestAGM101_TC10_EmptyBodyWithNilErrorDoesNotPanic`.
- **Hardcoded MongoDB Atlas credential** (FIXED, but **credential still needs
  rotating** - it is in git history). `internal/utils/store/store_test.go` and
  `internal/utils/archiver/integration_test.go` fell back to a live
  `mongodb+srv://` string with an embedded password when `MONGO_URI` was unset,
  no skip guard. That made `go test ./...` require network. Both now `t.Skip`.

## 3 pre-existing failures - FIXED 2026-08-24

Both were test-code bugs. Suite is now fully green (45 packages), and the CI
`test` job is `allow_failure: false`.

- **archiver, 2 subtests**: `Archiver.triggerCh` was never set in the struct
  literal, so `Run` selected on a **nil channel** (blocks forever) while the test
  sent on an unrelated local channel. Only the 2 subtests that needed `Run` to
  RECEIVE a trigger failed; 5 others call `ArchiveSources` directly, and
  "Context cancellation" only needs the `ctx.Done()` branch. Fix = wire
  `triggerCh` into the literal.
- **schema, TestSchemaEquivalency**: fixture bug, not impl bug. The two payloads'
  `tags` arrays held different element shapes (object vs string); the footprint
  walks into `arr[0]`, so one side contributed extra paths. Impl is CORRECT to
  distinguish them. Added `TestSchemaFootprintDistinguishesArrayElementShape` to
  pin that, plus `TestSchemaFootprintOnlyInspectsFirstArrayElement` documenting
  the real remaining gap: only `arr[0]` is walked, so a key appearing solely in a
  later element yields **no drift signal**.

## Infrastructure gaps closed

- Makefile had **no lint target at all**, yet `.gitlab-ci.yml` called
  `make lint-verify` in before_script. Added lint/lint-fix/lint-new/lint-verify
  mirroring parser-service.
- Neither repo ran tests in CI (lint stage only). Added `build` + `test` stages
  here, `test` stage in parser-service.
- `make test-live` runs `internal/providercontract/live_test.go` behind the
  `liveprovider` build tag - real calls to USGS, GVP, VolcanoDiscovery,
  AviationWeather, GWIS, NASA GEOS-5, FIRMS. Asserts response SHAPE only, never
  values. Excluded from CI by construction. FIRMS needs `FIRMS_MAP_KEY`.

## Gaps left by design

- gwis_ecmwf/meteofrance/nasa shell out to `python3 internal/script/*.py` partway
  through `fetch()`; tests stop at that boundary (so FWI danger-class thresholds
  and coordinate precision are untested).
- `internal/fire/copernicus` hardcodes its download URL and builds its own
  http.Clients that bypass the feed - not reachable from a test without refactor.
- weatherpipeline NOAA/ECMWF/DWD URLs are hardcoded consts; only seam is
  overwriting the unexported `Downloader.client` with a `roundTripFunc`.
