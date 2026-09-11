---
title: internal/app, internal/earthquake, internal/fire - code review findings
type: project
updated: 2026-09-07
---

# crawler-service - recall-biased review of internal/app + internal/earthquake + internal/fire (2026-09-07)

Reviewed the initial-commit diff (~7700 lines) for these three packages. Repo compiles
clean (`go build`/`go vet` both silent) so all findings are logic bugs, not compile
errors. Four high-confidence findings, ranked by severity:

1. **`internal/earthquake/shakemap/fetcher.go:248-253`** - `StoreIfAbsent` returns
   `(*store.DatabaseEnvelope, bool, error)` where the envelope is **always non-nil**
   on success (confirmed in `internal/utils/datafeed/feed.go:449-471`) regardless of
   whether the row was newly inserted. `processSingleShakeMapFeature` discards the
   `inserted` bool (`_`) and gates re-publish on `envelope != nil`, which is always
   true. Result: every polling cycle that sees an already-processed event (its
   `updated` timestamp gets bumped by unrelated USGS product changes, e.g. new DYFI
   submissions, for days/weeks after the quake) re-publishes the *same* unchanged
   ShakeMap products via `PublishBatchWithEventID` -> `datafeed.PublishBatch` ->
   `df.publisher.Publish` (a real downstream queue message each time). No test pins
   the `inserted=false` case. Fix: gate on `inserted`, not `envelope != nil`.

2. **`internal/earthquake/gdacs/fetcher.go:183-185` (`gdacsDate`)** - the GDACS
   earthquake list query only supports **day-granularity** `fromdate`/`todate`
   (confirmed by `fetcher_test.go` asserting `"2026-07-27"`/`"2026-07-28"`), but the
   cursor/incremental design copied from EMSC/USGS assumes sub-day precision narrows
   the window each poll. With the documented interval (10m, `config.example.yaml`),
   `start` and `end` fall on the same UTC date on almost every cycle, so the fetcher
   re-requests and unconditionally re-`Store`s + `PublishBatch`es the **entire day's**
   event list every ~10 minutes (plain `Store`, no id/dedup, so a fresh row each
   time) - i.e. continuous duplicate downstream publishes of the same earthquakes
   for as long as they stay within "today".

3. **`internal/fire/gwis_nasa/config.go:23`** - default (and `config.example.yaml:536`)
   `base_url` is `https://maps.effis.emergency.copernicus.eu/gwis`, copy-pasted from
   the sibling `gwis_ecmwf`/`gwis_meteofrance` WMS configs. But `gwis_nasa/fetcher.go`
   doesn't build a WMS request at all - it does
   `fmt.Sprintf("%s/%s/FWI.GEOS-5.Daily.Default.%s.nc", BaseURL, year, date)`,
   expecting a plain NASA GEOS-5 NetCDF file server. Neither `config.yaml` nor
   `config.docker.yaml` override it. If gwis_nasa is ever enabled following the
   example config, every fetch cycle 404s for all 7 lookback days and the source
   never produces data (`"could not find a valid NASA NetCDF file in the last 7
   days"`), silently (fetcher errors are just logged, not fatal).

4. **`internal/earthquake/usgs/fetcher.go:270-316` (`fetchDetails`) / same pattern
   in `shakemap/fetcher.go:152-184`** - all-or-nothing semantics: if even one of
   N concurrent per-event detail fetches errors, `errors.Join` propagates and the
   *entire* cycle's result (list envelopes + every other successfully-fetched
   detail) is discarded - `PublishBatch` is never called and the local cursor is
   never advanced. A single flaky upstream detail endpoint blocks all progress for
   that fetcher indefinitely (or until that one event stops erroring), instead of
   publishing the N-1 successes and retrying just the failure next cycle.

## Prior related finding (already fixed, see [[agm-crawler-test-coverage]])
`internal/fire/gwis_nasa` had a nil-deref (retry loop dereferenced a nil `err`)
that could panic the whole process since fetchers run as bare goroutines with no
recover in `internal/app/app.go`. Verified fixed in this diff (fetcher.go:101-106
now synthesizes a non-nil `fetchErr` for the empty-body-nil-error case). Checked
sibling `gwis_ecmwf`/`gwis_meteofrance` for the same pattern - both safe (`%v` on
possibly-nil err, not `.Error()`).

## Architecture notes worth remembering
- All earthquake fetchers (`usgs`, `emsc`, `gdacs`, `shakemap`) share one cursor
  pattern: `local.Get/Set` keyed by `<SourceName>.cursor`, storing `end` as
  RFC3339Nano and reading it back as `start = value - UpdateLookback`. SourceName
  constants are all unique across earthquake+fire packages (checked via grep) so no
  cursor-key collisions.
- `internal/app/app.go`'s `Run()` starts every fetcher as a **bare goroutine**
  (tracked by a local `sync.WaitGroup`, NOT the `errgroup`), each with its own
  `context.WithCancel(gctx)`. A fetcher's error is only logged, never fails the
  process - by design (one bad source shouldn't kill the crawler).
- Config `Validate()` is inconsistently called: some blocks in `app.go` validate
  before registering (`GDACS` flood, `IFRC`, `WorldTides`, `TropicalCyclone*`,
  `Atmospheric Water`, etc.), others don't (`USGSEarthquake`, `EMSCEarthquake`,
  `GDACSEarthquake`, `ShakeMapEarthquake`, all of `ActiveFire.*`). Not a bug per se
  - every fetcher's own `Run()` calls `f.config.Validate()` before its loop - just
  inconsistent style; a bad config surfaces as a logged error instead of a fast
  startup failure.
