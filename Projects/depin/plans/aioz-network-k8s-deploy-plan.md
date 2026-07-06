---
type: project
project: depin
title: AIOZ network — Kubernetes deploy plan (free/OSS stack)
status: active
tags: [aioz, depin, kubernetes, k3s, deployment, monitoring]
created: 2026-07-06
---

# AIOZ network — Kubernetes deploy plan (free/OSS stack)

Deploy plan for the depin/AIOZ network on a multi-node, self-hosted, 100% free/OSS stack.
Goal: spawn/scale any component across VPSs and monitor centrally.

**Stack:** k3s (multi-node) + KEDA (queue autoscaling) + kube-prometheus-stack
(Prometheus/Grafana/Alertmanager) + Loki/Promtail (logs). All free, self-hosted.

## Prerequisite (DONE)
- Modern k3s kubelet **refuses cgroup v1** — caused earlier crash-loop.
- **cgroup v2 now enabled** (`stat -fc %T /sys/fs/cgroup` → `cgroup2fs`). Unblocks multi-node k3s.
- k3d single-host is superseded by real multi-node k3s. Reinstall any half-installed k3s fresh.

## Component inventory (verified in code)

| Component | Invocation | Workload | Scale | Persistence | Ports | Identity |
|---|---|---|---|---|---|---|
| coord api | `coord api` | Deployment+HPA | horizontal | none | 7777 gRPC, 7779 P2P | shared |
| coord audit | `coord audit` | Deployment+KEDA | queue | none | debug only | shared |
| coord repair | `coord repair` | Deployment+KEDA | queue | none | debug only | shared |
| coord core | `coord core` | Deployment replicas:1 | singleton | none | debug | shared |
| coord ranged-loop | `coord ranged-loop` | Deployment replicas:1 | singleton | none | debug | shared |
| coord relay | `coord relay` | Deployment | horizontal | none | 7781, 7788 | shared |
| worker | **`aioznode api`** | StatefulSet | N, manual | **PVC 500GB+** | 7777 | **unique per pod** |
| edgeserver | `edgeserver run` | Deployment+HPA | horizontal | none | 8080 HTTP | shared cert |

### Critical code facts
- worker binary = `aioznode`; `run` is an **empty stub** → real process is **`aioznode api`**.
- worker finds coord via **`trust.sources`** list (not a single URL); unique identity per node; loses pieces if PVC lost.
- edgeserver needs **`coord-peer-url` = `<uuid>@host:port`** (required), stateless, `:8080`, `GET /download`.
- coord `core`/`run` migrate the DB; api/audit/repair/ranged-loop gate on `schema_migrations`.
- **Every** component debug server defaults to `127.0.0.1:0` (random) → must set
  `AIOZ_DEBUG_ADDR=0.0.0.0:9090` for `/health` + `/metrics`.
- All config injectable via `AIOZ_*` env (prefix `aioz`, `.`/`-`→`_`).

## Phases
0. cgroup v2 on every VPS (DONE).
1. Multi-node k3s: server on VPS-A (`--secrets-encryption --disable traefik`), agents join via node-token. Label storage nodes `role=storage`.
2. Image + registry: build in CI/clean box (`.dockerignore` excludes `dev/`, `**/data/`), push linux/amd64 to GitLab Container Registry, create imagePullSecret.
3. coord: external Postgres DSN secret; core migrates first, others gate on schema. (Existing manifests in `deploy/k8s/`.)
4. worker StatefulSet: `volumeClaimTemplates` (PVC/pod, local-path), initContainer runs `keytool create` per pod (idempotent), env `AIOZ_TRUST_SOURCES`, `AIOZ_STORAGE_PATH=/data/storage`, `AIOZ_CONTACT_EXTERNAL_ADDRESS`. Scale via `kubectl scale statefulset/worker`.
5. edgeserver Deployment+HPA: `AIOZ_COORD_PEER_URL`, shared identity secret, LB/Ingress `:8080`.
6. KEDA: `helm install keda kedacore/keda`; ScaledObject postgres scaler on audit/repair queue depth (scale 0→N by backlog). Verify real queue table names in `coord/db/migrations/*.sql`.
7. Monitoring: `kube-prometheus-stack` + `loki-stack` via Helm. PodMonitor selects `app.kubernetes.io/part-of: coord` (+worker/edgeserver) → auto-scrape `/metrics:9090`. Grafana dashboards (queue depth, repair rate, worker disk); Alertmanager (pod down, backlog, disk pressure).

## Identity — keytool
- keytool is now a **standalone binary** (`cmd/keytool`), removed from coord. Build separately: `go build -o bin/keytool ./cmd/keytool`.
- Command: **`keytool create <service> --identity-dir <dir> [--difficulty N]`** (default difficulty 36 = slow mining). Writes **4 files**: `ca.cert`, `ca.key`, `identity.cert`, `identity.key` (no `priv_key.json`).

## Docker image gotchas (learned the hard way)
- Runtime-only Dockerfile: `COPY bin/coord` + `COPY bin/keytool` (no build stage). Base ubuntu:24.04 (glibc 2.39). ENTRYPOINT `["coord"]`.
- `.dockerignore` MUST exclude `dev/`, `**/data/`, `**/hashstore/` — else build context balloons to ~90GB and fills disk. Keep `!bin/coord`, `!bin/keytool`.
- Image must be **linux/amd64** to match VPSs (`docker build --platform linux/amd64`).
- Transfer: `docker save | gzip | ssh ... docker load` (NOT `docker import` — strips entrypoint). Then `k3s` image import. `docker load` on a **full disk** silently corrupts the image → "executable not found".
- k8s `:latest` defaults to `imagePullPolicy: Always` → set `IfNotPresent` for locally-imported images (patch already in `deploy/k8s/base/kustomization.yaml`).

## Open items to confirm
1. Image registry URL + pull credentials (GitLab Container Registry).
2. Exact audit/repair queue table names for KEDA.
3. Postgres: external managed (recommended) vs in-cluster pinned StatefulSet.
4. Worker public reachability (public IP per node vs relay/NAT) → sets `contact.external-address` / `HavePublicAddress`.

## Related
- Manifests in repo: `deploy/k8s/` (base + overlays: prod/staging/local/postgres, examples).
- Full working plan file (session): `~/.claude/plans/humble-fluttering-biscuit.md`.
- Fallback if k8s ever off the table: Docker Compose per VPS (`docker-compose.coord-split.yml`) + standalone Prometheus/Grafana/Loki.
