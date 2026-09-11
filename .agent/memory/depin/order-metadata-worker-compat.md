---
type: fact
tags: [incident, orders, workers, compatibility, signing]
created: 2026-08-26
agent: main
---

On 2026-08-26, edge Loki showed systemic download failures: each retry received `PermissionDenied: invalid order limit ... signature is not valid` from every selected worker. This is coordinator order-limit signature verification, not verification using the per-limit Piece key. Likely cause is a mixed rollout after commit `8cb91d6` (2026-08-21): coordinator signing added `EncryptedMetadataKeyId` and `EncryptedMetadata` to `OrderLimitSigning`, while fleet workers built before that change re-encode only the old fields and therefore reject every new coordinator signature. Deploy workers that include the matching generated protobuf/signing changes; fresh limits must then be minted. A temporary coordinator rollback of signing those fields restores old-worker compatibility but makes contract metadata tamperable, so it is not a safe permanent fix.

Remediation started 2026-08-26: published fleet worker `v0.0.11` from source commit `21d7fb1` (which includes the compatible signing/protobuf fields) at `http://10.0.0.67:8821/worker-v0.0.11.zip`. Versioncontrol now suggests `v0.0.11` with the existing `safe_rate: 144` staged 0→100% rollout (about 10 minutes after the service restart at 03:57 UTC). The archive passed `unzip -t`, reports exactly `Version: v0.0.11`, and the file server recorded its first updater downloads immediately.

Follow-up live check at 04:17 UTC changed the primary diagnosis: reachable transcribe workers were already running `v0.0.11` and reported themselves up to date, while edge Loki still showed the same failures. The stale component is the deployed edge server: an edge binary generated before the Aug 21 `OrderLimit` schema update discards the unknown encrypted-metadata fields when decoding the coordinator's `DownloadManifest`, then forwards the altered limit to the worker. The worker's re-encoding consequently cannot match the coordinator signature. Rebuild/redeploy every edge replica with the Aug 21+ generated protos and go-sdk commit `73f0c70` (current monorepo source `21d7fb1` pins it), then make a fresh download request. No Piece-key rotation is required.

Fleet update activation is environment-gated, not inferred from versioncontrol: the container entrypoint runs the updater only when `ENABLE_AUTO_UPDATE=true`, and Compose must recreate containers after the value changes. The reachable transcribe fleet is at `/home/aioz/hub-v2-workers/depin-worker-fleet`; its containers already have `ENABLE_AUTO_UPDATE=true`, `VC_SERVER_ADDRESS=http://10.0.0.67:10000`, and a 1-minute check interval.
