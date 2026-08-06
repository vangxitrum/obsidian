---
type: index
project: depin
tags: [depin, index]
created: 2026-07-05
---

# DePIN (AIOZ hub)

Docs moved from `work/depin-workspace/depin` on 2026-07-05.

## Specs
- [[2026-05-22-uplink-sdk-design]]
- [[2026-05-25-uplink-sdk-upload-to-worker-design]]
- [[2026-05-25-uplink-sdk-upload-pipeline-refactor-design]]
- [[2026-06-04-vertical-split-restructure-design]]
- [[2026-06-04-worker-alias-design]]
- [[2026-07-14-contract-file-tags-usage-design]] — S3-faithful contract tags (bucket) shipped + file tags (object) deferred; usage-by-tag queries

## Plans
- [[2026-08-06-segment-verify|segment-verify (manual segment health checking)]] - standalone `cmd/segment-verify` binary ported from Storj `cmd/tools/segment-verify`: probes every holder of a segment with a 1-byte `Download` under a GET_AUDIT order limit and reports found / not-found / offline per piece. Read-only (no reputation writes, no repair enqueue). Bulk `run range|contracts|read-csv` with Storj's CSV outputs, plus targeted `check segment|file|contract` with a table and verdict exit code; handles RS and CLONE (CLONE threshold is 1). Library in `coord/segmentverify/`, queries reuse `auditSegmentScan`, probe reuses `audit.NewDialFetcher` + `ClassifyFetchError`. Key traps documented: order limits MUST be minted per probe (usedserials keys on coordinator+serial only, so a per-segment serial breaks the moment two pieces share a worker), `err == nil` with 0 bytes is not success, concurrency defaults to 64 not Storj's 1000 because of the shared-relay `MaxConnsPerIP` cap, and the revocation DB must default to `memory://` or the bolt flock collides with a live coordinator
- [[2026-08-04-testplanet-test-cases|Implement docs/test-plan.md with testplanet]] - turn the 159-row standing regression list into an executing suite: merge develop into feat/implement-testcases, add TC traceability columns, three harness extensions (relay peer + behind-NAT workers, in-planet versioncontrol, deposit chain fake), then one commit per feature section (~90 new testplanet tests); documents the five structural reasons testplanet cannot serve the remaining rows (one process/no exec, all peers on 127.0.0.1, no CLI layer, planet-boot cost vs pure functions, no non-Go systems), and flags that coord/deposit + cmd/worker-updater + versioncontrol/api are already near-fully covered by package tests. Phase 2b adds section 18 Deletion (DEL): 19 TC rows + 15 tests covering DeleteFile/DeleteContract end to end, and corrects three test-plan rows that name code the repo does not have (a `task.go` cleanup chore, a `file_storage_finalized` table, multipart upload)
- [[2026-08-04-component-version-tags|Component version tags]] - per-component build versions derived from each component's own git history (storj scripts/bake.sh model); exact repo tag makes every component ship that tag, otherwise `v0.0.0-dev.<ts>.g<hash>` per component; new `Source:` field carries the repo tag into every binary; scripts/component-version.sh + Makefile.build ldflags macro, versioncontrol untouched
- [[2026-08-04-delete-contract|DeleteContract (bucket-equivalent delete)]] - new StorageService.DeleteContract RPC mirroring Storj DeleteBucket: contract row is SOFT-deleted (deleted_at + status=DELETED) so tallies/egress-rollups/invoices stay billable, files+segments hard-deleted in a batched CTE, non-empty rejected with FailedPrecondition unless force=true, mark-then-drain so uploads cannot race in; plus go-sdk DeleteContract/DeleteFile wrappers and the GetByIDIncludingDeleted fix that keeps usage history readable after delete
- [[2026-07-30-client-usage-billing-onchain-deposits|Client usage billing, funded by on-chain AIOZ deposits]] - prepaid USD balance funded by per-client on-chain AIOZ deposits; period invoices debit balance from existing storage/egress meters; includes deposit watcher, ledger, CLI/RPCs, and Phase B gating notes
- [[2026-07-29-edge-download-speed|Edge download-speed improvements (steps 4/2/3)]] - pin the exact relay-reuse reset limit, then bounded relay connection reuse (raise worker rcmgr per-conn stream cap + warm K-conn pool + keepalive) and parallel/prefetch segments in go-sdk; edge download is latency-bound on per-piece connection setup over the 38ms WAN relay, not bandwidth/disk.
- [[2026-07-21-logship-loki-retention|Log shipping (Loki) + metrics retention]] - custom pkg/logship push client mirroring pkg/telemetry, hidden noflag config for the Loki URL/token delivered via a fleet-packaging secrets.yaml (env vars cannot reach noflag fields), Loki+Prometheus retention split (15d/25GB Prometheus, 15d/~75GB Loki)
- [[2026-07-14-contract-file-tags-usage|Contract + file tags usage]] - S3 bucket/object tag model + usage-by-tag queries; replaces the earlier download-ticket-tags approach
- [[2026-06-04-worker-vertical-split]]
- [[2026-06-05-coord-vertical-split]]
- [[aioz-network-k8s-deploy-plan|k8s deploy plan]] - full free/OSS multi-node k3s + KEDA + monitoring
- [[2026-07-06-aioz-k8s-deploy-runbook|k8s deploy runbook]] - 9-step zero→running sequence
- [[2026-07-22-worker-tags-port|Worker tags (node tags) port]] - foundation-first port of Storj node tags into depin: pkg/nodetag signing, worker advertises signed tags on check-in, coord verifies and stores them in a signed worker_tags table, keytool sign-tags mints authority-signed tags, trusted_node maps to worker.is_trusted; placement/selection consumption deferred to Phase 2.

## Component docs
- [[dataflow-upload]] · [[dataflow-audit]] · [[dataflow-payments]] · [[transport]] · [[configuration]]
- [[identity-trust]] · [[worker-autoupdate]] · [[piece-limit-encryption-fix]]
- [[COMPONENT_DOCS_CHECKLIST]]
- [[2026-07-06-deploy-staging|Deploy: Staging]] - staging k8s deploy of the coord stack
- [[2026-07-06-worker-onboarding|Worker Onboarding]] - full new-worker onboarding flow
- [[2026-07-21-relay-domain-locator-split-horizon-design]] — put a domain (not raw IP) in the worker relay locator + split-horizon DNS so edge/coord dial the relay over the free VPC; one reserve.go change preserves the /dns4/ domain into the signed order limit.
- [[2026-07-21-relay-domain-locator-plan]] — impl plan for the relay /dns4 locator + split-horizon DNS: Task1 reserve.go domain-preserving locator (TDD), Task2 config, Task3 authoritative split-horizon DNS + egress monitor, Task4 e2e.
- [[2026-08-04-direct-edge-worker-connections]] - two-phase plan to get edge/worker traffic off the relay.
