# parser-service — memory index

Backend datasource ports into parser-service. Each source = 4 files:
`internal/types/<t>.go`, `internal/datasource/<t>.go` (New<Camel>Handler), `internal/database/<t>.go` (Upsert...), `migrations/<NNN>_<table>.up/down.sql`.

## Pattern (copy from swpc_alert.go / water.go / drought.go)
- Process: rawref.Require -> rawref.ReadBytes -> checksumutil.VerifySHA256 -> decode (supports {status,body} fixture envelope; body may be JSON-quoted string) -> map -> store upsert -> ProcessResult kind:"database".
- Shared helpers only: metadataUint/metadataString (coral_reef_watch.go/water.go), sourceTimestamp (severe_weather.go). Prefix ALL other package-level idents with camelCase source name (datasource pkg is shared; marketstack has 4 siblings).
- DB layer uses Table("...")+map[string]interface{}+clause.OnConflict (NOT gorm models). dbutil.Nullable* for pointers. `interface{}` never `any`.
- Register in internal/app/process.go (do NOT edit from subagent).

## 2026-07 batch (migrations 077-086) — DONE
alphavantage_putcallratio(077), cftc(078, HTML report parse; report_type via metadata picks disaggregated/tff which sets spec=MM/comm=Prod or spec=LF/comm=Dealer), currencyratefrankfurter(079), fed(080; series_code/name via metadata; sync_states table skipped as fetcher-state), marketstack_commodity(081)/index(082; benchmark_id+weight via metadata)/liveprice(083; /intraday HTTP poll not websocket)/price(084; change computed vs MarketstackPricePrevAdjClose DB seed), polymarket_historyproxy(085; market via metadata clobTokenId)/hotmarkets(086; category via metadata, quota-allocation logic dropped).
- None were websocket-only; all HTTP-polled -> all persisted.
- Backend sources with no store.go (alphavantage in-memory, polymarket in-memory/LRU): designed new tables.
