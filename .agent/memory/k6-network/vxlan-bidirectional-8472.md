---
type: fact
tags: [k6-operator, k3d, flannel, networking, firewall, debugging]
created: 2026-07-17
agent: main
---

k6-operator TestRun stuck (runner pods stuck `ContainerCreating`, then once past
that, TestRun **STAGE frozen at `created`** with `2/2 runner pods ready` but no
starter) was a **one-way firewall on the flannel VXLAN port**.

**Symptom chain:** ContainerCreating (earlier, disk) -> solved -> then stuck at
`created`. Operator log: `failed to get status from hls-loadtest-service-1: Get
"http://10.43.x.x:6565/v1/status": dial tcp ...:6565: i/o timeout` +
`service is not ready`.

**Root cause:** VXLAN **8472/udp is stateless and needs BOTH directions opened
independently** on each host firewall. Only `control -> gen:8472` was allowed;
`gen -> control:8472` was blocked. So control->runner encapsulated packets
arrived, but the runner's replies (gen->control:8472) were dropped -> `i/o
timeout` (timeout, NOT "connection refused"). Operator could never confirm the
runner k6 REST API :6565, so it never launched the starter -> STAGE stuck.

**Key facts:**
- `:6565` = k6 REST API, **pod-network only** — NEVER a host firewall rule. It
  rides INSIDE the 8472/udp VXLAN. Operator hits it via the **Service ClusterIP
  (10.43.x)**, kube-proxy DNATs to the runner pod (10.42.x on gen).
- TCP flows (6443 API, 10250 kubelet) are stateful so one-way rules work; UDP
  8472 is not — needs explicit reverse rule.
- This box is a **DigitalOcean droplet** (`/mnt/volume_sgp1_15`): the **DO cloud
  firewall is separate from ufw** — 8472/udp must be allowed in BOTH, both boxes,
  both directions.

**Fix:** on control: `ufw allow from <GEN_IP> to any port 8472 proto udp` (+ same
in DO cloud firewall). After that the operator reconciles and STAGE advances
`created -> started -> running`.

**Gotcha when verifying:** `run-hls-case.sh` begins with
`kubectl delete testrun hls-loadtest --wait`, which SIGTERMs any in-flight runner
(`test run was aborted because k6 received a 'terminated' signal`, summary all
zeros, `vus max=0`). Re-running the script or ^C mid-run kills the current run.
Run ONCE and let it finish (~3 min: RAMP_UP 30s + HOLD 2m + RAMP_DOWN 30s).

**Multi-gen gotcha:** EVERY generator needs its own `gen->control:8472/udp`
allow rule, and gens can sit on DIFFERENT /24 subnets, so a single `/24` rule
silently misses off-subnet gens. Observed: control + `gen-uplink-test` on
`10.130.137.0/24` but `gen-stagging` on `10.130.119.0/24`. Use per-IP rules or a
wide `10.130.0.0/16`. A freshly wiped+rejoined gen may also get a NEW IP -> its
old firewall rule no longer matches. With N loadgen nodes, bump
`PARALLELISM=N ./tests/run-hls-case.sh` so one runner lands per gen.

Topology recap: controller `k3d-k6net-server-0` (10.42.0.x pods, tainted, priv IP
10.130.137.64) + generators `gen-stagging` (10.130.119.62) and `gen-uplink-test`
(10.130.137.45), both labeled `k6.io/role=loadgen`. See [[hls-load-test]]. Port
map is in RUNBOOK.md §Ports.
