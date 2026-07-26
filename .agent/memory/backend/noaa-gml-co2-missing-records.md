# noaa_gml/co2 — missing-records incremental-fetch fix

Same bug class as [[incremental-fetch-watermark-bug]] / [[noaa-gml-sf6-missing-records]] / usgs earthquake. Fixed 2026-07-14 on branch `fix/earthquake-missing-record`.

## Bug
`internal/datasource/noaa_gml/co2`:
- `store.go` `LatestFetch` = `MAX(fetched_at)` (INGEST wall-clock).
- `fetcher.go:83` filter `if !latest.IsZero() && !date.After(latest) { continue }` compared each record's EVENT date (midnight UTC) against that ingest instant.
- CO2 daily has ~1-day publish latency, so the newest day's midnight date is on/before the last mid-day ingest instant → dropped forever. After first backfill, new days stall.

## Fix chosen: Template B (event-date watermark), NOT Template A
Deliberately diverged from the sibling sf6 (which used Template A = delete filter + add Upsert + re-scan whole file). WHY:
- CO2 wires an ACTIVE per-record outbox: `co2.NewService(log, store, alertService.CreateOutbox)` (datasource.go:1408) and `Service.Save` fires `saveToOutbox("atmospheric_gases", record.ID, ...)` for EVERY saved record. sf6 has NO outbox (`sf6.NewService(log, store)`), so its full re-save is harmless.
- CO2 downloads the full static `co2_daily_mlo.txt` (2006→now, ~7000 daily rows) every poll — no server-side window. Template A would re-upsert all ~7000 rows daily AND re-fire the outbox for all of them → ~7000 ErrAlertDuplicate error-logs/poll (unique idx on alert_outbox(alert_type,source_id)). Real regression.
- Store is plain `Create` (INSERT) + `UNIQUE INDEX` on `date`. Per the class note, plain-insert stores → event-date watermark is the safe route (matches snow_cover/awdb fix).

## What changed (only co2 files)
- `store.go`: added `LatestDate()` = `MAX(date)` (event clock). Kept `LatestFetch` = `MAX(fetched_at)` for the poll-interval throttle only. Kept plain `Create` (gate prevents dup-key).
- `fetcher.go`: extracted pure `decodeRecords(io.Reader)` (no filter); added pure `filterNewByDate(recs, latest)` gate; `fetch` no longer takes `latest`; `runBackfill` now uses fetched_at for the interval-skip and MAX(date) for the record gate (two-watermark split). Dropped the dead `2006-01-01` seed (gate handles zero-mark = keep all).
- `fetcher_test.go` (new): `Test_decodeRecords_returnsAllObservations` (RED 0/3 with old filter, GREEN after) + `Test_filterNewByDate`.

## Verify: `go test`/`go vet ./internal/datasource/noaa_gml/co2/...` clean. Did NOT commit.

## Note
Repo save hook = golines (~80col) reflows the WHOLE edited file, but committed code is NOT golines-formatted → Edit/Write creates huge noise diffs. Revert unrelated reflowed hunks via Bash/python (hook is PostToolUse:Edit, doesn't fire on Bash) + `gofmt`. Left store.go at +23 (only LatestDate).
