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
- [[2026-07-21-logship-loki-retention|Log shipping (Loki) + metrics retention]] - custom pkg/logship push client mirroring pkg/telemetry, hidden noflag config for the Loki URL/token delivered via a fleet-packaging secrets.yaml (env vars cannot reach noflag fields), Loki+Prometheus retention split (15d/25GB Prometheus, 15d/~75GB Loki)
- [[2026-07-14-contract-file-tags-usage|Contract + file tags usage]] - S3 bucket/object tag model + usage-by-tag queries; replaces the earlier download-ticket-tags approach
- [[2026-06-04-worker-vertical-split]]
- [[2026-06-05-coord-vertical-split]]
- [[aioz-network-k8s-deploy-plan|k8s deploy plan]] - full free/OSS multi-node k3s + KEDA + monitoring
- [[2026-07-06-aioz-k8s-deploy-runbook|k8s deploy runbook]] - 9-step zero→running sequence
- [[2026-07-22-worker-tags-port|Worker tags (node tags) port]] - foundation-first port of Storj node tags into depin: pkg/nodetag signing, worker advertises signed tags on check-in, coord verifies and stores them in a signed worker_tags table, keytool sign-tags mints authority-signed tags, trusted_node maps to worker.is_trusted; placement/selection consumption deferred to Phase 2.

## Component docs
- [[dataflow-upload]] · [[dataflow-audit]] · [[transport]] · [[configuration]]
- [[identity-trust]] · [[worker-autoupdate]] · [[piece-limit-encryption-fix]]
- [[COMPONENT_DOCS_CHECKLIST]]
- [[2026-07-06-deploy-staging|Deploy: Staging]] - staging k8s deploy of the coord stack
- [[2026-07-06-worker-onboarding|Worker Onboarding]] - full new-worker onboarding flow
- [[2026-07-21-relay-domain-locator-split-horizon-design]] — put a domain (not raw IP) in the worker relay locator + split-horizon DNS so edge/coord dial the relay over the free VPC; one reserve.go change preserves the /dns4/ domain into the signed order limit.
- [[2026-07-21-relay-domain-locator-plan]] — impl plan for the relay /dns4 locator + split-horizon DNS: Task1 reserve.go domain-preserving locator (TDD), Task2 config, Task3 authoritative split-horizon DNS + egress monitor, Task4 e2e.
