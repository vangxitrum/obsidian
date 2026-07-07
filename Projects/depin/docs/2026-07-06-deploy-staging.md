---
type: doc
project: depin
tags: [depin, kubernetes, staging, deployment]
created: 2026-07-06
---

# Deploy: Staging

How to stand up a full `coord` stack in the **staging** environment on
Kubernetes. Staging runs exactly one instance of every role, no autoscaling,
with an in-cluster Postgres and placeholder secrets - a self-contained mirror of
prod topology at minimum scale.

Manifests live in `deploy/k8s`, packaged with Kustomize. See
`deploy/k8s/README.md` for the general layout and `README-vps.md` for a
single-VPS variant.

---

## 0. What staging is

The staging overlay (`deploy/k8s/overlays/staging/kustomization.yaml`):

- **Namespace** `depin-stag` (retargets every base resource; renames the base
  Namespace object).
- **Resources** = `base` + `postgres` (in-cluster DB) + `examples` (placeholder
  secrets).
- **One replica of every role.** `core`, `ranged-loop`, `audit`, `repair`,
  `relay` are already `replicas:1` in base; `api` defaults to 2 and is pinned to
  1 here. No HPAs (those live only in `overlays/prod`).
- **Image** `coord:latest` - override `newTag` to your staging image
  (`registry/coord:staging`).

Contrast with prod: prod uses an external managed Postgres, real image tags,
horizontal HPAs, and secrets created out-of-band (no `examples`, no in-cluster
`postgres`).

### Roles (from base)

| Role | Workload | Scale in staging | Notes |
|------|----------|------------------|-------|
| `core` | Deployment, `Recreate` | 1 (singleton) | **applies DB migrations on startup** |
| `ranged-loop` | Deployment, `Recreate` | 1 (singleton) | gates on schema-ready |
| `api` | Deployment (HPA off) | 1 | gRPC 7777, P2P 7779, LoadBalancer Svc |
| `audit` | Deployment (HPA off) | 1 | queue consumer (SKIP LOCKED) |
| `repair` | Deployment (HPA off) | 1 | queue consumer (SKIP LOCKED) |
| `relay` | Deployment | 1 | P2P 7781, status 7788, no DB |

No leader election in code - singletons are enforced by `replicas:1` +
`Recreate`. Only `core` migrates; every other DB role blocks in an initContainer
until `schema_migrations` is present and clean.

---

## 1. Prerequisites

### Cluster

Any k8s cluster with a working `LoadBalancer` provider (or use `port-forward`).
kind / k3d / minikube are fine for staging.

### Image

Build and make the image available to the cluster.

```bash
docker build -f Dockerfile.coord -t coord:latest .

# kind:
kind load docker-image coord:latest
# k3d:
k3d image import coord:latest
```

If you push to a registry instead, set it in
`overlays/staging/kustomization.yaml` under `images: newTag`. The base patch sets
`imagePullPolicy: IfNotPresent`, so a locally-loaded `coord:latest` is used
without pulling.

### Identity (shared, created once)

All roles share one read-only identity secret.

```bash
coord keytool new --identity-dir ./id     # -> identity.cert, identity.key, priv_key.json

kubectl -n depin-stag create secret generic coord-identity \
  --from-file=identity.cert=./id/identity.cert \
  --from-file=identity.key=./id/identity.key \
  --from-file=priv_key.json=./id/priv_key.json
```

> The `examples` overlay ships an `identity-secret-example.yaml` placeholder. For
> anything beyond a first smoke test, replace it with a real identity as above so
> the node ID is stable across redeploys.

### Secrets

Staging pulls placeholder `coord-db` / `coord-wallet` from `examples`. That is
enough to boot against the in-cluster Postgres. To point at a real DB, drop
`../postgres` + `../../examples` from the overlay and create the secrets
out-of-band:

```bash
kubectl -n depin-stag create secret generic coord-db \
  --from-literal=AIOZ_DB_POSTGRES_DSN='postgresql://user:***@HOST:5432/hub?sslmode=require'

kubectl -n depin-stag create secret generic coord-wallet \
  --from-literal=AIOZ_WALLET_PASSWORD='<strong-password>'
```

---

## 2. Deploy

```bash
# namespace is created by base/namespace.yaml (retargeted to depin-stag)
kubectl apply -k deploy/k8s/overlays/staging
```

Kustomize applies, in order: namespace -> ConfigMap -> in-cluster Postgres ->
example secrets -> `core` (migrations) -> `ranged-loop` -> horizontal roles
(`api`, `audit`, `repair`, `relay`) + their Services.

Boot ordering is enforced at runtime, not by apply order: every non-`core` DB
role blocks in an initContainer until `core` has applied migrations and
`schema_migrations` is clean.

---

## 3. Config knobs (all via `AIOZ_*` env)

From `base/configmap-common.yaml` (ConfigMap `coord-common`):

- `AIOZ_DEBUG_ADDR=0.0.0.0:9090` - health/metrics bind (overrides the default
  loopback so k8s probes + Prometheus can reach it).
- `AIOZ_SERVER_REVOCATION_DBURL=memory://` - in-memory revocation cache so every
  pod is stateless (no PVC, no exclusive file lock).

From secrets:

- `AIOZ_DB_POSTGRES_DSN` - Postgres DSN (Secret `coord-db`).
- `AIOZ_WALLET_PASSWORD` - wallet password (Secret `coord-wallet`).

Probes hit `GET /health` on port 9090; `/metrics` (Prometheus) is on the same
port.

---

## 4. Verify

```bash
kubectl -n depin-stag rollout status deploy/coord-core   # migrations applied
kubectl -n depin-stag get pods                           # all Ready
kubectl -n depin-stag logs deploy/coord-core             # confirm migration log

# health check
kubectl -n depin-stag port-forward deploy/coord-api 9090:9090
curl localhost:9090/health

# queue consumers dedup via SKIP LOCKED - scaling is safe even in staging
kubectl -n depin-stag scale deploy/coord-audit --replicas=2
```

Expected: `coord-core` reaches `Ready` first (after migrations), `ranged-loop`
and the horizontal roles follow once their initContainers see a clean schema.

---

## 5. Monitoring (optional)

`deploy/k8s/monitoring/` ships kube-prometheus + Loki values and a
`PodMonitor` (`podmonitor-aioz.yaml`) that scrapes `:9090/metrics`, plus
`alerts.yaml`. Apply these into the monitoring stack's namespace if staging runs
Prometheus.

---

## 6. Teardown

```bash
kubectl delete -k deploy/k8s/overlays/staging
# in-cluster Postgres PVC is retained by default; delete explicitly if needed:
kubectl -n depin-stag delete pvc --all
```

---

## Staging vs prod vs local

| | staging | prod | local |
|--|---------|------|-------|
| Namespace | `depin-stag` | `coord` | `coord` |
| Postgres | in-cluster | external managed | in-cluster |
| Secrets | placeholder (`examples`) | out-of-band | placeholder |
| `api` replicas | 1 (pinned) | HPA (horizontal) | base default |
| HPAs | none | yes | none |
| Image | `coord:latest` (override) | versioned tag | `coord:latest` |

---

*Sources: `deploy/k8s/overlays/staging/`, `deploy/k8s/base/`,
`deploy/k8s/overlays/postgres/`, `deploy/k8s/examples/`,
`deploy/k8s/monitoring/`, `deploy/k8s/README.md`.*
