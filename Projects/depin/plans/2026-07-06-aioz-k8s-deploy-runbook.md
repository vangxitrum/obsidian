---
type: project
project: depin
title: AIOZ k8s deploy — step-by-step runbook
status: active
tags: [aioz, depin, kubernetes, k3s, runbook, deployment]
created: 2026-07-06
---

# AIOZ k8s deploy — step-by-step runbook

Zero → running, in order. Companion to [[aioz-network-k8s-deploy-plan]].
Manifests live in the repo: `deploy/k8s/` (base + overlays + components + monitoring).

## Step 0 — Prep (once, each VPS)
```
stat -fc %T /sys/fs/cgroup                     # must say cgroup2fs
/usr/local/bin/k3s-uninstall.sh 2>/dev/null    # wipe old broken k3s if present
```

## Step 1 — k3s cluster
```
# VPS-A (server)
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --secrets-encryption --disable traefik" sh -
sudo k3s secrets-encrypt status                # Enabled
sudo cat /var/lib/rancher/k3s/server/node-token   # copy TOKEN

# VPS-B, VPS-C (agents)
curl -sfL https://get.k3s.io | K3S_URL=https://<VPS-A-IP>:6443 K3S_TOKEN=<TOKEN> sh -

# on VPS-A
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get nodes                              # all Ready
kubectl label node <storage-nodes> role=storage
```

## Step 2 — Image
```
docker build --platform linux/amd64 -f Dockerfile.coord -t <registry>/coord:v1 .
docker push <registry>/coord:v1
kubectl create namespace coord
kubectl -n coord create secret docker-registry regcred \
  --docker-server=<registry> --docker-username=<u> --docker-password=<tok>
```
Set that image in each `kustomization.yaml` (`images:`).

## Config model — mounted config.yaml (NOT env)  ⚠️ important
Config is a **mounted `config.yaml`** with the EXACT flat-dotted flag keys from
`dev/<component>/config.yaml` — do NOT rely on `AIOZ_*` env vars: the flag paths include
struct prefixes, so the obvious env name is wrong and silently ignored:
- coord DSN key = `app-config.db.postgres-dsn` → env would be `AIOZ_APP_CONFIG_DB_POSTGRES_DSN`
  (NOT `AIOZ_DB_POSTGRES_DSN` — using that made coord fall back to the `:5445` default).
- worker keys are prefixed `worker.` (`worker.storage.path`, `worker.trust.sources`, `worker.debug.addr`).
- edgeserver keys are top-level (`coord-peer-url`, `server.listen-addr`, `debug.addr`).

coord: `coord-config` **Secret** (has DSN) mounted at `/data/config.yaml` (patch in base kustomization).
worker/edgeserver: `*-config` **ConfigMap** (no secrets) mounted at `/data/config.yaml`.
Per-pod-only values stay as env (worker `AIOZ_WORKER_CONTACT_EXTERNAL_ADDRESS` = host IP).

## Step 3 — Secrets + config
```
# identity (4 files from: keytool create <svc> --identity-dir ./id)
kubectl -n coord create secret generic coord-identity \
  --from-file=ca.cert --from-file=ca.key --from-file=identity.cert --from-file=identity.key
kubectl -n coord create secret generic edgeserver-identity --from-file=ca.cert ...

# coord config.yaml (contains DSN + wallet pw -> Secret). Keys = real flag paths.
cat > config.yaml <<'EOF'
app-config.db.postgres-dsn: postgresql://admin:admin123@postgres:5432/hub?sslmode=disable
wallet.password: <strong-password>
debug.addr: 0.0.0.0:9090
revocation-dburl: memory://
audit.enabled: true
EOF
kubectl -n coord create secret generic coord-config --from-file=config.yaml=./config.yaml
```
(staging bundles `coord-config`/`worker-config`/`edgeserver-config` from examples — only identity is yours.)

## Step 4 — coord roles
```
kubectl apply -k deploy/k8s/overlays/prod
kubectl -n coord rollout status deploy/coord-core     # migrations run here
kubectl -n coord get pods                             # api/audit/repair leave Init once core up
```

## Step 5 — workers
```
# fill worker/configmap.yaml trust-sources first
kubectl apply -k deploy/k8s/components/worker
kubectl -n coord get statefulset worker               # pods Ready, each PVC bound
```

## Step 6 — edgeserver
```
# fill edgeserver/configmap.yaml coord-peer-url first
kubectl apply -k deploy/k8s/components/edgeserver
kubectl -n coord get svc edgeserver                   # LB IP on :8080
```

## Step 7 — KEDA (queue autoscaling)
```
helm repo add kedacore https://kedacore.github.io/charts && helm repo update
helm install keda kedacore/keda -n keda --create-namespace
kubectl -n coord apply -k deploy/k8s/components/keda
# then remove hpa-audit/hpa-repair from prod overlay (KEDA owns audit/repair scaling)
```

## Step 8 — Monitoring
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts && helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace -f deploy/k8s/monitoring/kube-prometheus-values.yaml
helm install loki grafana/loki-stack -n monitoring -f deploy/k8s/monitoring/loki-values.yaml
kubectl apply -f deploy/k8s/monitoring/podmonitor-aioz.yaml
kubectl apply -f deploy/k8s/monitoring/alerts.yaml
```

## Step 9 — Verify
```
kubectl get nodes                                  # all Ready
kubectl -n coord get pods                          # all Running
kubectl -n coord scale statefulset/worker --replicas=3   # scale test
kubectl -n coord get scaledobject                  # audit/repair KEDA active
kubectl -n monitoring get svc monitoring-grafana   # open Grafana, dashboards live
```

## Placeholders to fill
- `<registry>` image path + pull creds
- coord node id / trust URL → `worker/configmap.yaml`
- coord uuid → `edgeserver/configmap.yaml` (`<uuid>@host:port`)
- real secret values (DSN, wallet password, identity files)

## First-run caveats
- worker `contact.external-address` = pod host IP; must be routable, else use relay/NAT (`HavePublicAddress=false`).
- edgeserver `run` needs `config.yaml`; initContainer runs `edgeserver setup` — confirm it reads `AIOZ_COORD_PEER_URL` from env in your build.
- worker identity initContainer mines difficulty 24; raise to coord's minimum if registration rejected.
