# Updater wedged by leftover staging file (2026-09-10, bug/worker-trash)

## Symptom
v0.0.12 rollout: 23/50 transcribe workers updated, 27 stuck forever on the 08-05 image binary. versioncontrol, fleet-fs and the rollout cursor were all fine (every host 200 on /versions; aioz5090 50/50).

## Root cause
worker-updater stages the new binary at `<binary>.<version>` (`/data/bin/worker.v0.0.12`) via `unpackBinary` with `O_CREATE|O_EXCL` (`cmd/worker-updater/binary.go`), then renames it into place. Cleanup happens only on error paths the updater itself sees. The user recreated the transcribe containers mid-rollout (09:46 and 10:24 UTC); updaters killed between create and rename left the file on the persistent `/data` volume, and every later poll failed with `open /data/bin/worker.v0.0.12: file exists` - forever. All 27 stuck had the file (12 complete 59.8MB, 15 partial 4.8-50.7MB); none of the 23 updated did. Storj's storagenode-updater has the identical O_EXCL pattern (binary.go:106), so this wedge is upstream too.

## TRAP that cost an hour
The updater logs `New version is being rolled out but hasn't made it to this node yet` every minute - that line is its SELF-update check (`"service": "worker-updater"`), NOT the worker's. The worker's real failure is the separate `ERROR Error updating service ... file exists` line. Misreading it sent me chasing the rollout cursor / safe_rate / versioncontrol code. Always grep `loop.go:48 Error updating service` first.

## Fix (uncommitted)
`unpackBinary` removes an existing target before the O_EXCL create (only the updater writes `<binary>.<version>`, so it can only be a leftover), logs "Removed leftover binary from an interrupted update." Test: `TestUpdate_LeftoverStagingFileFromInterruptedUpdate` in `cmd/worker-updater/update_test.go` (RED with the exact production error before, GREEN after; all 15 updater tests pass).

## Shipping caveat
Fleet containers run `/usr/local/bin/worker-updater` from the IMAGE (08-05), not the auto-updated data volume - the fix reaches the fleet only via an image rebuild (or updater self-update, which is lost on the next recreate). Stuck nodes need a manual `rm -f /data/bin/worker.v<ver>` until then.

Related: [[worker-storage-numbers]], [[fleet-host-tag]]
