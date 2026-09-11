# GC verified working end-to-end (2026-09-08)

Investigated "is GC working" against the live fleet. Verdict: **yes, all stages**. No bug.

## Confirmed numbers
- `gc_bloomfilter_bundles_built` 13,240 (+731/24h); observer finishes every ~2.5 min, `errored:false`, ~40.5k pieces/filter, 101 workers.
- `gc_sender_retain_success` 26,074 (+2,400/24h) = 101 workers x 24 hourly rounds. `gc_sender_retain_fail` +24/24h = ONE permanently dead worker (`1d98a0fb-9884-e09d-18e7-9837dc2b2e01`, "failed to dial"), not systemic.
- `(*Store).compactOnce` field=`total` 1592, successes 1540, **errors 52 (all within 24h)**, panics 0. 165 calls/24h.
- `(*Store).rewriteRecord` total 10,252,668, ALL successes, +1,513,571/24h. Reclaim is real and continuous.
- Live proof from `finished compaction` stats: worker-27 s0 LenLogs 3.3 GiB -> 2.5 GiB, DataReclaimed 0.9 -> 1.7 GiB. worker-30 s0 3.3 -> 3.1 GiB. TrashPercent runs 40-65%.

## THE TRAP that produced two wrong conclusions
`docker logs` on a fleet worker spans only **~76 minutes** (worker-1: 07:10 -> 08:26) even though the container has been up since Sep 3. Logs rotate hard.
- Grepping a 76-min window and seeing one Retain => wrongly concluded "workers only get a bloom filter once a day". One hourly retain in 76 min is CORRECT.
- Grepping the same window for "compact" and finding nothing => wrongly concluded "compaction has never run". Compaction is scheduled at `avgMinutes = 12h` (`worker/pkg/hashstore/db.go:558`), ~once per store per day.
**Always check the first timestamp of `docker logs` before concluding anything is absent.**

## Second trap: monkit field names
`function{name="(*Store).compactOnce", field="count"}` returns 52 - that is NOT the call count. Real fields: `total`, `successes`, `errors`, `failures`, `panics`, `recent`, `ravg`, `min`, `max`, `sum`, `current`, `highwater`. Use `field="total"` for calls. Querying `field="count"` silently gives you a different number, not an error.

## Where to look (all from the local box)
- Prometheus `127.0.0.1:9090`, Loki `:3100`, Grafana `127.0.0.1:3001` - the live coord/edge push here via remote_write + logship.
- Worker fleet: `ssh transcribe` (10.0.0.106), 50x `fleet-transcribe-worker-N` on `depin-worker:latest`. Worker data: `/data/storage/hashstore/<coord-id>/{s0,s1}`, received filter persisted at `/data/storage/meta/<coord-id>.bloomfilter` (mtime = last retain, updates hourly).

## Open items (not fixed)
- 52 `compactOnce` errors in 24h across 38 of 100 workers, all appearing within the last day. Error text not captured (rotated away). `error_name` label is empty.
- **Worker logs are not shipped to Loki** - `app` label has only `coord` and `edgeserver`. This is the root reason the investigation was so error-prone.
- No counters for bytes reclaimed / trash size; the numbers only exist inside the `finished compaction` log line's `stats` blob.
- `coord/gc/sender` ships hourly but the bloomfilter observer rebuilds every ~2.5 min, so ~29 of every 30 bundles built are discarded unsent.

## 2026-09-10 follow-up: "trash never reclaimed" (branch bug/worker-trash) - NOT a bug
Dumped worker-18 s0's live hashtbl (docker cp -> local `hashstore.OpenTable` + `Range`): 74,804 live, 92,829 trash, **0 trash past its expiry**. Trash expiry days were 09-11/09-12/09-13/09-17 only - every earlier batch had already expired and been dropped. Trash lives exactly `Compaction.ExpiresDays` = 7 (hidden, Storj default), then is dropped at the next compaction and its bytes freed when the log is rewritten. Fleet-wide ~50-65% of LenLogs is trash simply because the fleet deletes a lot and holds it 7 days.
- These are WORKER settings (verified via `worker api --help`, note the `worker.` prefix): `--worker.storage.hash-store-config.compaction.expires-days` (hidden, default 7), `...compaction.alive-fraction` (0.25, log rewrite eagerness), `...compaction.delete-trash-immediately` (hidden). Env form: `AIOZ_WORKER_STORAGE_HASH_STORE_CONFIG_COMPACTION_EXPIRES_DAYS` (prefix `aioz`, `.`/`-` -> `_`, pkg/process/exec_conf.go:191). Fleet containers only set `WORKER_*` env today.
- Expired trash frees bytes only when its log is rewritten: rewrite probability per compaction = ((0.25/0.75)*(1-alive)/alive)^2 -> 100% at <=25% alive, ~11% at 50%, ~1% at 75%; unexpired trash counts as alive; ~320 MiB alive rewritten per compaction cap (RewriteMultiple 10 x hashtbl).
- TRAP (2026-09-11): the `~/gc-capture/poll.sh` compaction capture on transcribe UNDERCOUNTS by >2x - one pass of `docker logs --since 65s` over 50 containers takes ~85 s, then sleeps 60 s, so each 145 s cycle only sees a 65 s window. Ground truth for "when did each store last compact" = BIRTH time (`stat -c %W`) of the highest-numbered `/data/storage/hashstore/<coord>/{s0,s1}/meta/hashtbl-*` - compaction creates a new file with a new id. NOT mtime: the live table is written in place on every piece insert, so mtime tracks the last upload (that mistake produced a false "83/100 compacted"; birth time said 39/100, matching the coord DB exactly: 5 workers with both stores compacted = 5 workers with trash < 50 MB).
- Demo coord DB read works via a throwaway client container (host has no psql; `docker exec depin-pgtest psql` to the demo DB was classifier-blocked, `docker run --rm postgres:18.1-alpine psql "<url>"` with `PGOPTIONS='-c default_transaction_read_only=on'` was allowed after the user asked). Group fleets by `worker_tags` name='host' (`convert_from(value,'UTF8')`); 2026-09-11 03:09 UTC: transcribe trash 109.5 GB / used 260.6, oldwin (7-day control) 165.0 / 293.2. Prometheus `compactOnce` increase is also useless here (restarts create new series; 33 series for 50 workers).
- END-TO-END CORRECTNESS TRACE (2026-09-11, worker-18 / alias 478): exported all live remote segments (root_piece_id, pieces, created_at) read-only from the demo DB, decoded pieces with `file.Pieces.Scan`, derived per-worker IDs `root.Deriver().Derive(workerUUID.Bytes(), num)` (same as `coord/gc/bloomfilter/observer.go:228` and `coord/order/signer.go:250`), compared with the worker's copied hashtbls + bloom filter taken 8 s apart. Result: 67,752 referenced pieces, ALL held live; 0 referenced pieces trashed; 0 missing; 0 referenced pieces absent from the filter for segments older than the scan start (1,876 absent were created after the scan - expected); unreferenced: 0.418 GB >72h awaiting compaction, 0.377 GB <72h skew-protected, 4.049 GB already trash; bloom false positives 7/22,166 = 0.03%. Verdict: worker trash logic is correct; no data-loss path observed. Re-run on ALL 50 transcribe workers (export 04:24:46 UTC, copies within 30 s): 3,409,635 referenced pieces, all held live; 0 referenced trashed, 0 missing, 0 filter false negatives on pre-scan segments (4,470 post-scan, expected); unreferenced: 23.3 GB >72h awaiting compaction, 19.4 GB <72h protected, 105.7 GB already trash; false positives 0.03%. Copy all workers in one `tar -czf - | tar -xzf -` ssh stream (1.9 GB) rather than per-file scp.
- Compaction gate (`db.go:604-612`): a worker compacts only if at least one store's last compaction/table-creation day is before today UTC; one store per run, ~1/720 chance per minute, forced after 24 h (timer resets on process restart).
- `DELETE_TRASH_IMMEDIATELY=true` verified: every compaction after enabling it ended with Trash=0 B. Disk bytes lag behind: dead space stays in logs until later compactions rewrite them (~320 MiB alive per run).
- `.restore` file on workers is from 2026-08-07, so `restored()` (trash with expiry <= restore+7) no longer masks anything.
- `DataReclaimable` right after a compaction is tiny BY DEFINITION (it just rewrote); it does not measure trash.
- Record-level dump is the only way to prove trash ages - stats blobs only give totals.
- ~/gc-capture/poll.sh on transcribe (PPID 1) still running, capturing compaction lines to ~/gc-capture/compaction.log.

Related: [[coord-gc-metrics-instrumentation]], [[coord-missing-committed-ttl-gc]], [[logship-grafana-allvalue-bug]]
