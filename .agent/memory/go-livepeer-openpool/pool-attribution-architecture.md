---
type: fact
tags: [go-livepeer-openpool, livepeer, architecture, payouts, events]
created: 2026-08-25
agent: main
---

What the openpool fork adds on top of go-livepeer (branch
`origin/release/open-pool/v0.8.10`). One sentence: **it makes a single Livepeer
Orchestrator able to run as a mining pool, by giving each remote worker an on-chain
identity and emitting a per-job event stream an off-chain pool manager can settle from.**

## 1. ETH address as remote-worker identity
Upstream identifies a remote transcoder / AI worker only by connection address.
The fork adds `bytes ethereum_address = 4` to both `RegisterRequest` and
`RegisterAIWorkerRequest` in `net/lp_rpc.proto`, threaded through
`server/ot_rpc.go` -> `core/orchestrator.go serveTranscoder/Manage/RemoteTranscoder`
and `core/ai_orchestrator.go` for AI workers.

- Standalone worker/transcoder now **hard-requires `-ethAcctAddr`**:
  `starter.go` does `glog.Fatal("Starting a pool-based clients require an ethereum address")`.
- Orchestrator side rejects registration with `errNoEthAddress` if the field is nil.
  => a stock upstream transcoder cannot join a pool orchestrator, and vice versa.

## 2. Event tracker (`events/tracker.go`, new package)
`events.GlobalEventTracker` singleton, `CreateEventLog(eventType, k, v, ...)`.
Buffered in memory, flushed to **S3** as JSON batches keyed
`<region>/<nodeType>/<region>-<nodeType>-<start>_<end>.json`.
Config is **env-var only, not flags** (`NewPoolTrackerOptionsFromEnv` in starter.go):
`POOL_S3_HOST`, `POOL_S3_BUCKET`, `POOL_S3_ACCESS_KEY`, `POOL_S3_SECRET_KEY`,
`POOL_REGION`, `POOL_NODE_TYPE` (all required), plus optional
`POOL_EVENT_THRESHOLD` (default 10000) and `POOL_FLUSH_INTERVAL_SECONDS` (default 60).
Transcoder/AIWorker node types get a `NoopEventTracker` (console only) - only the
orchestrator/gateway actually ships events.

Event types emitted: `orchestrator-reset` / `gateway-reset` / `node-reset` (startup),
`worker-connected`, `worker-disconnected`, `job-received`, `job-processed`,
`ticket-redeemed` (from `pm/sendermonitor.go`, includes faceValue/winProb/txHash).

## 3. Per-job payout accounting
- Transcoding: `core/orchestrator.go OnTranscode()` runs async after every successful
  remote transcode; sums encoded pixels, multiplies by `node.GetBasePrice(sender)`
  (falls back to `"default"`), computes `realTimeRatio = duration/responseTime`, emits
  `job-processed` with the worker's ethAddress. `TranscodeData` gained a
  `ResponseTime` field just for this.
- AI: `server/ai_http.go` emits `job-processed` with `requestID`, pipeline, modelID,
  outPixels, price. `RemoteAIWorker.Process` gained a `requestID` param purely so the
  pool manager can **correlate `job-received` (has ethAddress) with `job-processed`
  (has fees)** - the AI path splits attribution across two events, unlike transcoding.

## 4. Local event mirror + HTTP feed
`common/db.go` adds a `pool_events` table (id/payload/version/dt) with
`CreatePoolEvent` / `FindPoolEvents(since)`, exposed on the CLI webserver as
`GET /pool/events?lastCheckTime=<RFC3339>` (`server/handlers.go poolEventsHandler`).
No pagination, no purge - both flagged as TODO in the source.

## 5. Webhook-based worker selection (AI only, partial)
`-remoteWorkerWebhookURL` flag -> `RemoteAIWorkerManager.webhookURL`.
`fetchPreferredWorkers(pipeline, modelID)` GETs it and expects `[{"ethAddress": ...}]`;
`selectWorker` filters live workers to that set, **falling back to the unfiltered list
on any error or empty response**. The equivalent for transcoders is still a TODO
comment. This is the hook for pool policy (stake/reputation-based routing).

## 6. The segment is the unit of attribution (sizing note)

Segmentation is gateway-only and happens before anything reaches an orchestrator
(RTMP -> LPMS `SegmentRTMPToHLS` with `SegLen = 2 * time.Second`, a hardcoded const
at `server/mediaserver.go:63` with **no CLI flag**; HTTP-push callers arrive
pre-segmented; the AI/live path uses a separate shelled-out `ffmpeg -f segment` into
named FIFOs at `media/rtmp2segment.go:80`). Orchestrators and transcoders never
segment - they get one segment file and produce all N renditions from a single
ffmpeg invocation (`core/transcoder.go:440-465`), which is why the sanity check is
`len(tSegments) != len(md.Profiles)`.

So one segment = one payment = one `job-processed` event = one payout row. At
`SegLen=2s` each stream emits 0.5 events/sec, so ~1000 concurrent streams is ~500
events/sec. Against the default `POOL_EVENT_THRESHOLD=10000` that flushes to S3 about
every 20s, i.e. **the threshold rather than the 60s timer governs at scale** - check
S3 PUT cost before running a large pool.

Related: [[fork-delta-lives-on-release-branches]] for how to read the diff,
[[codec-support-and-verification-map]] for what is (and is not) verified.
