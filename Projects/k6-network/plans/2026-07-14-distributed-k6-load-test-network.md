# Plan: Distributed k6 Load-Test Network (5 VPS, k3s + k6-operator, external Prometheus)

## Context
The team needs a repeatable, distributed load-testing platform to hammer an internal HTTP
API from several machines at once. A single k6 process caps out at one box's CPU/network.
Goal: run **one** `TestRun` manifest that fans a single k6 script across 4 load-generator
VPS, streaming all metrics to the team's **existing external Prometheus/Grafana** for live,
aggregated dashboards.

Decisions locked with the user:
- **Orchestration:** k3s (lightweight Kubernetes) + **k6-operator** (v1.5.0). One manifest
  auto-splits a test across N pods via k6 execution segments.
- **Metrics:** k6 native `experimental-prometheus-rw` **pushes** to an **existing external
  Prometheus** (off the 5 VPS); Grafana on that same host visualizes. k6 has no scrape/pull
  endpoint (runner pods are ephemeral) — push/remote-write is the only native path.
- **Roles:** 1 VPS = k3s server (control-plane, **tainted** — no load pods); 4 VPS = k3s
  agents (generators). No monitoring stack runs on the 5 VPS.
- **Target:** internal HTTP API/web app.

All syntax below verified July 2026 (k6-operator v1.5.0, CRD `k6.io/v1alpha1` `kind: TestRun`,
k3s stable, k6 `experimental-prometheus-rw` output).

## Architecture
```
   ┌──────────────── EXISTING external host (NOT one of the 5 VPS) ───────────────┐
   │  Prometheus  (remote-write receiver ENABLED, :9090/api/v1/write)              │
   │  Grafana     (dashboard 19665 "k6 Prometheus")                               │
   └───────────────────────────────▲──────────────────────────────────────────────┘
                                    │ remote_write (push, HTTPS+auth if over public net)
                                    │ from each generator node IP (pod egress SNAT'd to node IP)
   ┌──────────── control VPS (k3s server, tainted) ────────────┐
   │  k3s control-plane + k6-operator (initializer/starter/CRD) │  ── schedules runner pods ──┐
   └───────────────────────────────────────────────────────────┘                             │
        ┌──────────────────────┬──────────────────────┬──────────────────────┐               │
   ┌────┴─────┐          ┌─────┴────┐            ┌─────┴────┐           ┌──────┴───┐  ◄─────────┘
   │ gen VPS1 │          │ gen VPS2 │            │ gen VPS3 │           │ gen VPS4 │  k3s agents,
   │ k6 runner│──push──▶ │ k6 runner│──push──▶   │ k6 runner│──push──▶  │ k6 runner│  k6.io/role=loadgen
   └──────────┘          └──────────┘            └──────────┘           └──────────┘
   parallelism:4 → each pod owns 1/4 of the VUs (execution-segment split), pushes to external Prometheus
```

## Repo layout (this repo: k6-network)
```
k6-network/
├── README.md                          # runbook: bootstrap → run → view
├── ansible/                           # provision k3s across the 5 VPS (idempotent, versioned)
│   ├── inventory.ini                  # [control] 1 host + [generators] 4 hosts
│   ├── site.yml                       # k3s server+agents, node labels, firewall, sysctl/ulimit tuning
│   └── roles/{k3s_server,k3s_agent,firewall,tuning}/
├── platform/
│   ├── install-operator.sh            # helm install grafana/k6-operator (only cluster add-on)
│   └── external-prometheus.md         # how to enable remote-write receiver on the existing Prom host
├── tests/
│   ├── scripts/api-test.js            # k6 script, parametrized by __ENV (BASE_URL, VUS, DURATION)
│   ├── configmap.yaml                 # or `kubectl create configmap ... --from-file`
│   ├── rw-auth.secret.example.yaml    # optional: basic-auth/bearer for the remote-write endpoint
│   └── testrun.yaml                   # TestRun: parallelism=4, nodeSelector, prometheus-rw env → external URL
└── grafana/
    └── README.md                      # import dashboard 19665 on the external Grafana
```

## Implementation steps

### 1. Prereqs on all 5 VPS
- Reachable private IPs, unique hostnames (`control`, `gen1..gen4`).
- Open firewall **between the 5 nodes** (keep off public internet):

  | Proto | Port | Scope | Purpose |
  |---|---|---|---|
  | TCP | 6443 | agents → server | k3s API |
  | UDP | 8472 | 5 nodes ↔ | Flannel VXLAN |
  | TCP | 10250 | 5 nodes ↔ | kubelet |

- **Cross-host egress:** on the **external Prometheus host**, open `9090/tcp` (or the TLS proxy
  port) inbound **from the 4 generator node IPs**. Pod egress is SNAT'd to the node IP, so
  Prometheus sees traffic from the 4 generator VPS addresses.
- Generator OS tuning (matters at high VU counts): raise `nofile` ulimit, widen
  `net.ipv4.ip_local_port_range`, enable `net.ipv4.tcp_tw_reuse`. Applied by ansible `tuning` role.

### 2. Stand up k3s (ansible/site.yml recommended; manual one-liners as fallback)
- **control VPS (server):**
  ```bash
  curl -sfL https://get.k3s.io | sh -
  sudo cat /var/lib/rancher/k3s/server/node-token    # join token
  kubectl taint node control node-role.kubernetes.io/control-plane=:NoSchedule   # keep load off it
  ```
- **4 generator VPS (agents):**
  ```bash
  curl -sfL https://get.k3s.io | K3S_URL=https://<CONTROL_IP>:6443 K3S_TOKEN=<node-token> sh -
  ```
- Label the 4 agents for the TestRun `nodeSelector`:
  ```bash
  kubectl label node gen1 gen2 gen3 gen4 k6.io/role=loadgen
  ```
- Verify: `kubectl get nodes -o wide` → 5 Ready (1 control tainted, 4 loadgen).

### 3. Prepare the EXTERNAL Prometheus + Grafana (existing host — platform/external-prometheus.md)
Enable the remote-write receiver on the existing Prometheus so k6 can POST to it:
- **Prometheus binary/systemd:** add flag `--web.enable-remote-write-receiver` and restart.
- **kube-prometheus-stack (if that's what runs there):** set
  `prometheus.prometheusSpec.enableRemoteWriteReceiver: true`.
- k6 will target: `http://<EXT_PROM_HOST>:9090/api/v1/write`.
- **Security (do this if the hop crosses the public internet):** don't expose raw 9090.
  Front it with a TLS reverse proxy + basic-auth or bearer token, OR put the generators and the
  Prometheus host on a private network / WireGuard. k6 supports `K6_PROMETHEUS_RW_USERNAME`/
  `_PASSWORD`, `_BEARER_TOKEN`, and TLS vars — see step 5.
- Grafana (same host): import dashboard **19665** ("k6 Prometheus").

### 4. Install k6-operator on the cluster (platform/install-operator.sh)
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install k6-operator grafana/k6-operator          # (or: kubectl apply -f bundle.yaml)
```
- Verify: operator pod Running; `kubectl get crd | grep k6.io` → `testruns.k6.io`.

### 5. Author the test (tests/)
- `api-test.js`: HTTP flow against `__ENV.BASE_URL`; put **TOTAL** desired VUs in `options`
  — the operator auto-divides by `parallelism` (200 VUs @ parallelism 4 → 50/pod). Do NOT pre-divide.
- ConfigMap (`tests/configmap.yaml`) holding the script.
- Optional `tests/rw-auth.secret.example.yaml`: a Secret with remote-write creds, referenced via
  `runner.env[].valueFrom.secretKeyRef` (keeps tokens out of the manifest).
- `tests/testrun.yaml`:
  ```yaml
  apiVersion: k6.io/v1alpha1
  kind: TestRun
  metadata:
    name: api-loadtest
    namespace: default
  spec:
    parallelism: 4
    arguments: --out experimental-prometheus-rw --tag testid=run-001
    script:
      configMap: { name: k6-test, file: api-test.js }
    runner:
      nodeSelector: { k6.io/role: loadgen }
      env:
        - { name: K6_PROMETHEUS_RW_SERVER_URL, value: "https://<EXT_PROM_HOST>:9090/api/v1/write" }
        - { name: K6_PROMETHEUS_RW_TREND_STATS, value: "p(95),p(99),min,max,avg" }
        - { name: K6_PROMETHEUS_RW_PUSH_INTERVAL, value: "5s" }
        # if the endpoint is secured (recommended over public net):
        # - { name: K6_PROMETHEUS_RW_USERNAME, valueFrom: { secretKeyRef: { name: rw-auth, key: username } } }
        # - { name: K6_PROMETHEUS_RW_PASSWORD, valueFrom: { secretKeyRef: { name: rw-auth, key: password } } }
        # - { name: K6_PROMETHEUS_RW_INSECURE_SKIP_TLS_VERIFY, value: "false" }
      resources:
        requests: { cpu: "500m", memory: "512Mi" }
        limits:   { cpu: "1", memory: "1Gi" }
  ```

### 6. Run & observe
```bash
kubectl apply -f tests/configmap.yaml -f tests/testrun.yaml
kubectl get pods -w    # api-loadtest-initializer → 4 runners (one/gen node) → starter releases in sync
```
- External Grafana → dashboard **19665**: watch aggregate RPS / p95-p99 / error rate summed
  across the 4 pods (series tagged `testrun_name`).
- Re-run: `kubectl delete -f tests/testrun.yaml`, edit `testid`, re-apply.

## Mechanics worth knowing
- **Load split = execution segments.** Operator sets `--execution-segment(-sequence)` per pod;
  script VUs/iterations auto-divide by `parallelism`.
- **Initializer pod** validates the script first (`k6 archive`); if it fails, no runners start.
- **Runners start `--paused`; a starter Job** resumes all 4 simultaneously → synchronized load.
- **Series tag `testrun_name`** (operator ≥ v1.4.0, replaced `job_name`); dashboard 19665 already matches.
- **Egress SNAT:** runner pods reach the external Prometheus as their node's IP — that's why the
  firewall allow-list is the 4 generator node IPs, not a pod CIDR.

## Verification (end-to-end)
1. `kubectl get nodes` → 5 Ready (1 control tainted, 4 `k6.io/role=loadgen`).
2. Reachability: from a debug pod on a generator, `curl -v https://<EXT_PROM_HOST>:9090/api/v1/write`
   → connects (405/400 on GET is fine; proves network + firewall + TLS work before a real run).
3. Smoke TestRun (e.g. 40 VUs, 1 min) → exactly 4 runner pods, one per generator
   (`kubectl get pods -o wide`).
4. External Grafana 19665 shows non-zero aggregated RPS + latency during the run → proves the
   full remote-write path from all 4 pods to the external Prometheus.
5. Scale up: bump total VUs, re-apply; confirm ~1/4 load per node and the target API sees ~4×
   a single-node run.

## Notes for execution
- Provisioning path: **Ansible** (in-repo, idempotent) is the recommended default; the manual
  k3s one-liners in step 2 are the quick-start fallback.
- After approval: mirror this plan into the Obsidian vault (`Projects/k6-network/plans/`) via the
  hermes CLI, and record the setup decisions in agent memory (per standing global rules).
```
