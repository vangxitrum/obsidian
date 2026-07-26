---
type: decision
tags: [k6, k3s, k6-operator, ansible, prometheus, load-testing]
created: 2026-07-14
agent: main
---

Repo `/home/tuan/work/k6-network` scaffolded per the approved plan
`Projects/k6-network/plans/2026-07-14-distributed-k6-load-test-network.md`
(k3s + k6-operator v1.5.0 across 5 VPS, k6 `experimental-prometheus-rw` push to an
**existing external** Prometheus/Grafana, 1 tainted control node + 4 `k6.io/role=loadgen`
generator nodes).

This first pass was **scaffold-only**: every file from the plan's Repo layout was created
with placeholder values (`<CONTROL_IP>`, `<EXT_PROM_HOST>`, `<GEN1_IP>`..`<GEN4_IP>`,
`<SSH_USER>`, `<PATH_TO_SSH_KEY>`) - no SSH to real hosts, no `kubectl apply`/`helm install`
against a live cluster.

**Why:** confirmed with the user (AskUserQuestion) that no real VPS inventory/SSH details
were available in the repo at scaffold time - live provisioning was explicitly out of scope
for this pass.

Sub-decision resolved during scaffolding: the plan's firewall port table (step 1) didn't
name a tool. Asked the user; they chose **ufw** (not firewalld/iptables/none). The
`ansible/roles/firewall` role targets ufw specifically - it will need rewriting if the
target VPS are RHEL-family (firewalld) instead of Ubuntu.

**How to apply:** before running `ansible-playbook -i inventory.ini site.yml` for real, fill
in `ansible/inventory.ini` (control + 4 generator IPs, SSH user/key, external Prometheus
host) and the `<EXT_PROM_HOST>` in `tests/testrun.yaml`. If the target VPS aren't
Ubuntu/ufw-based, the firewall role needs to be swapped to firewalld/iptables first.

**Update 2026-07-14:** user asked to run this on k3d. Clarified k3d itself only clusters
containers on one Docker engine (shared bridge network + serverlb) - it can't span 5
separate physical hosts, so it would collapse the whole point of the plan (4 real,
separate CPU/network/egress-IP generators) onto one box. User confirmed intent was
per-VPS containerization, not collapsing onto one host, so `k3s_server`/`k3s_agent` roles
were rewritten to run **dockerized k3s** instead (`rancher/k3s` image via docker-compose,
`network_mode: host`, one container per VPS) rather than the native `get.k3s.io` install -
still 5 real machines/egress paths, same firewall/tuning/k6-operator/testrun setup
unchanged. New pieces: `docker-compose.server.yml.j2` / `docker-compose.agent.yml.j2`
templates, a shared `k3s_cluster_token` inventory var (static, since there's no easy
cross-container node-token file read like the bare-metal version had), and the
kubeconfig now lands at `/etc/rancher/k3s/kubeconfig.yaml` (not `k3s.yaml`) via bind mount.

**Update 2026-07-14 (2):** user filled real inventory values and asked to run ansible
itself in Docker too (`ansible/Dockerfile` + `ansible/docker-compose.yml`, `pip install
ansible` base image, mounts repo + SSH key, `ANSIBLE_HOST_KEY_CHECKING=False`). Built and
`--syntax-check`'d it against the real inventory - this surfaced two real bugs from the
earlier dockerization pass, now fixed:
1. Inventory group `[control]` containing a host also named `control` -> ansible
   name-collision warning. Renamed the **group** (not the user's host) to `k3s_control`
   across `inventory.ini`, `site.yml` (`hosts:`), and the agent compose template
   (`groups['k3s_control'][0]`).
2. `site.yml`'s inline "label generator nodes" / "show cluster status" tasks still called
   bare `kubectl --kubeconfig /etc/rancher/k3s/k3s.yaml ...` - leftover from before
   dockerization, host has no such binary/path anymore. Fixed to
   `docker exec k3s-server k3s kubectl ...`, matching the k3s_server role's taint task.

Also flagged for the user (not yet confirmed fixed): their `external_prometheus_host`
value included an `https://` scheme prefix, but that var feeds `ufw`'s `to_ip:` (needs
bare host/IP) and the same bare value is meant to go into `tests/testrun.yaml`'s
`<EXT_PROM_HOST>` placeholder (already wrapped in `https://...:9090/...` there).
(Fixed by the user shortly after - `external_prometheus_host` is now the bare host.)

Also wrote `RUNBOOK.md` (repo root) - the full copy-paste step-by-step, separate from
`README.md`'s architecture/overview. Keep both in sync when the ansible/compose mechanics
change; RUNBOOK.md is the one that goes stale fastest.

**Update 2026-07-14 (3):** user confirmed ansible itself runs *on the control VPS*
(not a laptop). Set `ansible_connection=local` on the `[k3s_control]` host in
`inventory.ini` so its own tasks execute locally instead of SSH-looping to itself;
only the 4 generators are still reached over SSH. Caveat documented in RUNBOOK.md: this
only works cleanly with ansible installed natively on the control VPS (Option A). If
using the dockerized ansible runner (Option B) on the control VPS with this local
connection, the control host's `docker compose up` etc. would run *inside the ansible
container* unless `/var/run/docker.sock` is bind-mounted in - RUNBOOK.md's Option B now
includes that mount. If ansible ever runs from elsewhere again (laptop, a 6th box),
`ansible_connection=local` must be removed from that inventory line first.

**Update 2026-07-14 (4):** user asked for a no-ansible path. Added `scripts/` -
`setup-control.sh`, `setup-generator.sh`, `label-generators.sh` - plain bash
translations of the ansible roles (firewall/ufw, tuning, dockerized k3s server/agent,
taint/label), meant to be scp'd + run by hand, one script per VPS role, as root. Vars
(IPs, token, prom host) are placeholders at the top of each script, same values as
`ansible/inventory.ini`. **These now need to be kept in sync by hand with the ansible
roles** - there's no shared source of truth between them (duplicated logic, not
templated from the same place). If the ansible roles change again, remember to mirror
the change into `scripts/*.sh` too, or flag the drift to the user.

**Update 2026-07-14 (5):** user asked to run this "with k3d" again and to clean up
RUNBOOK.md, removing unrelated options. Same terminology mix-up as update (1) - not
literal k3d (still can't span hosts), still means the dockerized-k3s-per-VPS setup
they already approved. Rewrote `RUNBOOK.md` from a 3-option sprawl (native ansible /
ansible-in-docker+socket-mount / no-ansible scripts) down to ONE linear path: native
ansible, run from the control VPS (matches the already-set
`ansible_connection=local`). The ansible-in-docker and `scripts/*.sh` paths still
exist in the repo and work, just aren't documented in RUNBOOK.md anymore - point the
user back to `ansible/docker-compose.yml` / `scripts/` directly if they ask for those
again.

**Update 2026-07-14 (6):** immediately after (5), user asked to actually use
ansible-in-docker as THE path (not native pip), and mentioned k3d is already
installed on their VPS. Clarified: having the k3d CLI installed doesn't remove its
single-Docker-host limitation - irrelevant to the multi-host goal either way, no
architecture change from it. Swapped RUNBOOK.md's step 1 from native
`pip install ansible` to the dockerized ansible runner (`ansible/Dockerfile` +
`docker build`/`docker run`, already built earlier in this session), with the
`/var/run/docker.sock` bind mount required because of `ansible_connection=local` on
the control host. RUNBOOK.md is now: dockerized-ansible, run from the control VPS,
dockerized k3s per node - the one canonical path going forward unless the user
changes it again.

**Update 2026-07-14 (7):** user asked to fix a gap I'd flagged earlier: since control
is tainted and nothing tolerated it, the k6-operator controller + each TestRun's
initializer/starter pods were landing on a generator node instead of control (CRD
confirms via `crd.md`: initializer inherits the runner's nodeSelector - i.e. loadgen -
if left unset; starter has the same nodeSelector/tolerations fields). Fixed by pinning
both to control instead, via the `node-role.kubernetes.io/control-plane` node label
(auto-set by k3s on server nodes) + a matching toleration:
- New `platform/k6-operator-values.yaml`, applied via `helm install -f` in
  `install-operator.sh` (also fixed a stale `k3s.yaml` kubeconfig path comment there
  while touching the file - should be `kubeconfig.yaml`).
- `tests/testrun.yaml` gained `spec.initializer`/`spec.starter` blocks with the same
  nodeSelector+toleration.
**Not verified against a live cluster** - the exact Helm value paths
(`nodeSelector`/`tolerations` at chart root) are the common convention but weren't
confirmed against the actual `grafana/k6-operator` chart's `values.yaml` (not present
in this repo, only `crd.md` which is CRD-only, not the Helm chart). Flagged in the
values file itself: run `helm show values grafana/k6-operator | grep -A3
nodeSelector` before applying to confirm the path, adjust if the chart nests it
differently.

**Update 2026-07-14 (8):** user asked to drop `network_mode: host` from the k3s
compose files (concern turned out to be a port conflict on the VPS, not security).
Flagged before implementing: without host networking, flannel (k3s's overlay CNI)
can't build a working VXLAN tunnel, since the container only sees an internal docker
bridge IP, not the VPS's real routable one - even with `--node-ip` overrides. This
would likely break the starter Job's unpause call to runner pods (cross-node pod IP
traffic) - test would hang paused. Better fix once port/service identified: keep
`network_mode: host`, disable the specific conflicting built-in k3s component instead
(most likely candidate: Traefik on 80/443, via `--disable=traefik` on the k3s server
command - unused here anyway, no ingress in this design). **Not yet implemented** -
still waiting on the user to confirm which port/service actually conflicts.

Separately, user hit a real bug running the dockerized-ansible path for real: SSH
identity file error (`no such identity: /home/root/.ssh/k6`) plus `control` failing
over SSH despite `ansible_connection=local`. Root cause: `ansible_ssh_private_key_file`
in `inventory.ini` must be the path as seen INSIDE the ansible container (the
right-hand side of the `docker run -v host:container:ro` mount), not the host-side
path - the user's inventory.ini had the host path. Fixed inventory.ini's key path to
`/root/.ssh/k6` (matching RUNBOOK's mount example) and added two gotchas to
RUNBOOK.md. Also worth checking next time this comes up: the user's *real* control
VPS inventory.ini may drift from this repo's copy (different IPs seen in their error
output - 10.130.137.64/10.130.119.62 - vs this repo's 10.130.137.48/10.130.137.32) -
this repo is not necessarily in sync with whatever they're actually running on the
VPS.

**Update 2026-07-14 (9):** playbook got further (firewall+tuning ran) but failed on
"Allow generator egress" with ufw's `ERROR: Bad destination address` - confirmed the
`external_prometheus_host`-is-a-hostname problem flagged back in update... (the
`ufw`/`community.general.ufw` `to_ip` field needs a literal IP, doesn't resolve
hostnames). Turned out the user manages all port access via the **DigitalOcean Cloud
Firewall UI** already (their VPS are DO droplets) and asked to drop ufw entirely
rather than patch it. Implemented:
- `ansible/roles/firewall/tasks/main.yml` gutted to a single `debug` reminder task -
  no more ufw. Port table kept as reference in `ansible/roles/firewall/defaults/main.yml`
  (unused now, comment explains why).
- `scripts/setup-control.sh` / `setup-generator.sh` (the no-ansible path) - same ufw
  blocks removed, replaced with a comment pointing at RUNBOOK.md.
- `RUNBOOK.md` gained a new "0.5 Configure the DigitalOcean Cloud Firewall" section
  with the exact inbound rule table (6443/tcp, 8472/udp, 10250/tcp from the other 4
  Droplet IPs) and an outbound note (9090/tcp to the Prometheus IP, only if outbound
  is already locked down - DO's default is open outbound).
**User is on DigitalOcean** - worth remembering for any future infra/networking
questions on this project (Cloud Firewall UI, not ufw/iptables, is the access-control
layer here).

**Update 2026-07-16:** resolved the pending port-conflict question from update (8).
User chose to **disable Traefik**, not drop host networking. Added
`--k3s-arg "--disable=traefik@server:*"` to the controller's `k3d cluster create` in
RUNBOOK.md §1 (frees the host's 80/443 that k3s Traefik binds under `--network host`;
nothing here uses ingress). Flag applies only at server start -> needs a cluster
recreate; verify `kubectl get pods -A | grep -i traefik` returns nothing. Host
networking KEPT (still required for flannel VXLAN / cross-node runner traffic).
**Generator side needs NOTHING** for Traefik: it's a k3s *server*-only addon and
generators are k3s *agents* (`command: agent --node-ip=...`) - the `--disable` flag is
server-only and agents don't run the addon manager, so disabling on the controller
kills Traefik cluster-wide. To make the disable a persistent default (user recreates
the cluster often), added **`k3d/create-controller.sh`** - one command that bakes in
host-net/no-lb/tainted-control-plane/`--disable=traefik`, writes the kubeconfig, and
prints the node-token; RUNBOOK.md §1 now references it and the k3d/README lists it.
There is NO controller-create script otherwise (RUNBOOK §1 was manual copy-paste; the
old `scripts/setup-control.sh` from the pre-k3d ansible era is gone after the k3d
migration commit).
Also fixed RUNBOOK §3: k6-operator no longer ships `releases/latest/download/bundle.yaml`
(404). Correct install = raw pinned URL
`raw.githubusercontent.com/grafana/k6-operator/v1.5.0/bundle.yaml` (`--server-side`),
or the helm path in `platform/install-operator.sh`. See [[hls-load-test]] for the HLS
test that runs on this cluster.
