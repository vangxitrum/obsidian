---
type: decision
tags: [k6, hls, m3u8, load-testing, aioz, streaming]
created: 2026-07-15
agent: main
---

Added an HLS/m3u8 load test to `/home/tuan/work/k6-network`:
`tests/scripts/hls-load-test.js` + `tests/scripts/playlist.m3u8` (sample VOD manifest,
197 signed-ticket segments against `http://68.183.189.51:8888/download?ticket=...&expire=...`
- an AIOZ-style download node).

**What the script does:** each VU = one viewer that parses the media playlist and
walks the segments, downloading each (binary). Env-driven per repo convention
(`example-load-test.js` style): `VUS`/`RAMP_UP`/`HOLD`/`RAMP_DOWN`, `REALTIME`
(true = pace by each segment's EXTINF duration/`RATE`, false = hammer back-to-back),
`MAX_SEGMENTS`, `START_RANDOM`, `SEG_TIMEOUT`. Custom metrics: `hls_segment_duration`,
`hls_segment_ttfb`, `hls_bytes_downloaded`, `hls_segment_failed`, `hls_expired_responses`.
Segment requests carry a constant `name:'hls_segment'` tag - **required** because each
URL has a unique ticket, so without it every request becomes its own Prometheus series
(cardinality blowup).

**The "pass a new file when links expire" requirement** (the tickets die in minutes):
two mechanisms - (1) file mode `-e PLAYLIST=/path/to/fresh.m3u8` (read via `open()` at
init, resolves rel. to script dir), (2) URL mode `-e PLAYLIST_URL=https://.../master.m3u8`
(fetched fresh in `setup()` each run, tickets always current). At start it decodes the
first segment's `expire=` param and LOUDLY `console.warn`s in file mode if already expired.

**Verified end-to-end** (not just syntax): ran via `docker run --rm -v .../tests/scripts:/s
-w /s grafana/k6 run hls-load-test.js -e VUS=1 -e MAX_SEGMENTS=2 -e REALTIME=false ...`
(k6 not installed natively; docker is). 6 segments, 100% HTTP 200, 8.9 MB pulled,
~660ms avg / ~515ms TTFB - live endpoint worked, tickets were valid until 2026-07-16
08:37 UTC. **The bundled playlist.m3u8 will be expired by then** - swap it before any
real run.

**Knee-finder mode** (`-e PROFILE=steps`): script builds `options` dynamically - one
sequential `constant-vus` scenario per level in `STEP_VUS` (default 10,20,30,40,50),
`STEP_SECONDS` each (default 60 -> 5m total), staggered via `startTime` so they never
overlap, each tagged `{step:N}` with its own `hls_segment_duration{step:N}: p95<SEG_P95_MS`
threshold. In the summary the KNEE = first step line that prints ✗. `STEP_ABORT=true`
stops at the first crossing. Default `PROFILE=ramp` = original warmup/hold/rampdown.
`SEG_P95_MS` (default 5000) parametrizes the p95 threshold in both modes. Verified both
option-trees build via `k6 inspect` (init-only, NO load sent): steps -> 5 scenarios +
5 per-step thresholds; ramp -> `viewers`/ramping-vus. First real 50-VU run on the cluster
already crossed p95<5000 (node is throughput-bound ~4.7 MB/s aggregate), which is why the
user wanted the knee finder.

**k6-operator / k3d path NOW WIRED** (`tests/hls-testrun.yaml`). The operator mounts the
WHOLE script ConfigMap as one dir, so bundling both files in a ConfigMap named `k6-hls`
makes `open('./playlist.m3u8')` resolve in the runner pod. Build the ConfigMap from the
two source files (rebuild = the expiry-swap workflow):
```
kubectl create configmap k6-hls \
  --from-file=hls-load-test.js=scripts/hls-load-test.js \
  --from-file=playlist.m3u8=scripts/playlist.m3u8 \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f tests/hls-testrun.yaml
kubectl logs -l k6_cr=hls-loadtest -f
```
For always-fresh tickets set `PLAYLIST_URL` env in the TestRun instead (each runner pod
fetches the manifest in its own setup(), so URL mode is fine with parallelism>1). TestRun
pins initializer/starter to control (tainted) + runner to loadgen, same as
`tests/testrun.yaml`. See [[scaffold-decisions]] for the cluster.

**Verified the risky bit:** `k6 archive hls-load-test.js` bundles `playlist.m3u8` into the
tar (`file/s/playlist.m3u8`) - i.e. the initializer's `k6 archive` will carry the manifest
to the runner. Confirmed via dockerized k6.

**Gotcha found while verifying:** `k6 archive` FREEZES `export const options` (VUS, stage
durations) at archive time from the env present THEN; passing `-e VUS=... -e HOLD=...` to
`k6 run <archive.tar>` does NOT shrink an already-baked options block. So don't two-step
archive-then-run with different env for a quick smoke (I accidentally ran the full
50-VU/2m profile against the live endpoint that way). Under the operator it's a non-issue:
the initializer archives WITH the runner env set, so options compute right. For a local
smoke, run the .js directly with `-e` (not a pre-built archive).
