# parser-service: Cloudflare Radar sources port

Backend `radarcloudflare/*` sources ported into parser-service (module gitlab.internal/aioz-map/parser-service).

## Pattern (copied from swpc_alert.go / water.go)
Per source: 4 files
- internal/types/<target>.go — record struct (from backend model.go Record)
- internal/datasource/<target>.go — New<Camel>Handler(store, logger); Source()==target; Process = rawref.Require -> ReadBytes -> checksumutil.VerifySHA256 -> decode({status,body} fixture envelope then Radar API {result,...}) -> map -> store Upsert -> ProcessResult kind:"database"
- internal/database/<target>.go — Upsert<Camel>Records (map[string]interface{} rows + clause.OnConflict, dedup in handler via recordMap keyed by conflict columns)
- migrations/NNN_<table>.up/down.sql

## Key insight: single-raw-payload
Backend fetchers loop multiple endpoints combining into one save, BUT parser processes ONE Radar API JSON payload per event; the discriminating `type` (inference_models / target / mitigation / device_type ...) comes from event metadata via metadataString(evt.Metadata,"type") with a per-source default. Same idea as water state_code from metadata. So multi-endpoint backend sources are still single-raw-payload for the parser -> none skipped.

## Rules followed
- interface{} not any; ALL package-level idents prefixed (radarAIBotSummary*, radarRobotsTxt*, radarAttackLocation*, radarAttackMitigation*, radarHTTPSummary*, radarRankingInternetService*) because they share package `datasource`.
- shared helpers metadataString (water.go), metadataUint (coral_reef_watch.go), stringutil.FirstNonEmpty — reuse, don't redefine.
- Register in internal/app/process.go (NOT app/); publisher cmd/publish-raw-event/main.go is generic (raw_ref + metadata), no per-source code.

## My 6 sources (source => table => migration#)
- radarcloudflare_aibot_ai_bot_summary => radar_cloudflare_ai_bot_summary => 058 ; conflict (type,name)
- radarcloudflare_aibot_robotstxt => radar_cloudflare_robots_txt => 059 ; conflict (type,name); has fully/partially cols
- radarcloudflare_attack_location => radar_cloudflare_attack_target_location => 060 ; conflict (type,rank,target_country_alpha2)
- radarcloudflare_attack_mitigation => radar_cloudflare_attack_mitigation => 061 ; conflict (type,name)
- radarcloudflare_http_summary => radar_cloudflare_http_summary => 062 ; conflict (type,name)
- radarcloudflare_ranking_internet_service => radar_cloudflare_ranking_internet_services => 063 ; conflict (service)

Existing migrations went up to 021 before this batch; verified in commit as of 2026-07-06.
