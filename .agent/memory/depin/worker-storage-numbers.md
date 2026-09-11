# Worker storage numbers audit (2026-09-10, branch bug/worker-trash)

Question: are the worker storage numbers right? Verdict: **usage numbers are right, free-disk numbers are wrong (always 0) and allocation is never capped by the real disk.**

## Correct
- Worker self-reported `used_space` matches the real disk: transcribe-worker-18 reported 6,318,934,336 vs `du -sb /data/storage` 6,319,080,310 (146 KB diff). Per store: s0 LenLogs 3.2 GiB + 32 MiB table vs du 3.28 GiB; s1 2.6 GiB vs 2.61 GiB. `HashStoreBackend.SpaceUsage` (LenLogs+TableSize, LenSet-LenTrash, LenTrash) is sound.
- Existing hashstore DBs are opened at startup (`backend.go` ReadDir in NewHashStoreBackend), not lazily - so used=0 is not a restart artifact.
- aioz5090 fleet (10.0.0.164) reports used_space=0 for all 50 workers: TRUE, they have no `/data/storage/hashstore` dir at all - zero uploads in 47h (likely selection starvation, see [[direct-conn-test-blockers]] new-node-fraction note). Not a numbers bug.

## BUG: storageStatus never populated
`worker/pkg/space/shared.go` `SharedDisk.DiskSpace` declares `var storageStatus StorageStatus` and never fills it. Storj's `storagenode/monitor/shared.go` fills it from `s.dir.AvailableSpace(ctx)` (DiskTotal/DiskFree) and also caps `available` at `DiskFree` (`if DiskFree > 0 && DiskFree < available`). Depin's port dropped both. Consequences:
- `Total` and `Free` are always 0 -> worker console `/api/v0/storage` total/free = 0, coord `workers.disk_space_free`/`disk_space_total` = 0, admin UI `view.DiskFree = w.DiskSpace.Free` (coord/admin/worker.go:143) = 0.
- Allocation is never clamped to real disk, available never clamped to real free space. **transcribe host: 13.8 GB free on the shared LV (99% used), but its 50 workers claim 189.9 GB available** (10GB ALLOCATED each). Disk-full on the host will hit before any worker thinks it is full.
- coord `SelectedWorker.FreeDisk` is copied from the row but read by nobody - coord has no free-disk selection gate; only the worker-side `AvailableSpace` check protects uploads, and that is what is overcommitted.

## FIXED (2026-09-10, bug/worker-trash, uncommitted)
Ported Storj's disk probe: `worker/pkg/space/dirspace.go` (DirSpaceInfo, 1-min cached, on `worker/pkg/du` statfs - `platform.DiskInfo` has no total), `SharedDisk.DiskSpace` fills storageStatus, caps allocated at DiskTotal and available at DiskFree. `HashStoreBackend.LogsPath()` added; `NewSharedDisk` now returns an error.
- SECOND bug found only by the e2e: the hashstore logs dir is created lazily on first write, so statfs failed on a fresh worker and it reported zeros even after the fix. Storj does `os.MkdirAll(logsPath)` in NewSharedDisk - ported.
- `PreFlightCheck` deliberately NOT ported: default MinimumDiskSpace is 500GB, fleet allocates 10GB -> Storj's check would stop every worker from starting.
- Tests: `internal/testplanet/disk_space_report_test.go` (1000 TB allocation -> total/free > 0, caps hold, coord row matches; RED before) + `worker/pkg/space/shared_test.go` (cap unit tests, mutation-proven).
- On a shared host the DiskFree cap only bites near full: transcribe workers each have ~3.7GB allocation left vs 13.8GB disk free, so numbers don't shrink today - the 500GB-of-allocations-on-a-13.8GB-disk is fleet config.
- Demo coord DB reads from this box are blocked by the auto-mode classifier even when the user asks; hand them a read-only SQL file instead.

## Worker live bytes ~1.5x coord tally = GC lag, NOT a bug (2026-09-11)
Coord `worker_storage_tallies` (sum of derived piece sizes) vs worker-reported `disk_space_used_for_pieces`: transcribe 81.5 vs 124.7 GB, oldwin 77.4 vs 120.1 GB; every worker 1.24-2.01x. Offline check on worker-18 (copied hashtbls + `/data/storage/meta/<coord>.bloomfilter`, applying the worker's own rule `filter.created - piece.created > MaxTimeSkew && !filter.Contains`): live 2.43 GB = in-filter 1.65 GB (coord tally 1.62 - match) + garbage >72h old 0.42 GB (trash is only MARKED at compaction, so it waits) + garbage <72h old 0.36 GB (protected by `MaxTimeSkew` default 72h, `worker/retain/retain.go:25`). High upload-then-delete churn (746 GB settled PUT vs 159 GB held) makes this window large.
- Method: throwaway Go tool using `piecepb.Unmarshal` -> `RetainRequest{Filter, CreationDate}`, `bloomfilter.NewFromBytes`, `hashstore.OpenTable` + `Range`; record Created is day-granular (`DateToTime`) exactly as the worker uses it.

## Traps
- `used_space` etc. are monkit IntVal -> Prometheus series per `field` (count/sum/min/max/ravg/recent). `sum(used_space)` across fields is garbage (gave 1.36e17). Always filter `field="recent"`.
- Live coord DB read via psql to 68.183.189.51 was blocked by the auto-mode classifier - ask the user before querying prod.
- oldwin (10.0.0.163) ssh: publickey denied from this box.

Related: [[worker-disk-capacity-findings]], [[worker-console]], [[gc-verified-working]]
