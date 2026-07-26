# Portable source-less worker package (`fleet.sh package`)

**What:** turnkey zip to run depin worker nodes on any amd64 machine with **only Docker**
(no repo, no Go, no make). Built 2026-07-19, verified e2e, not committed.

## Build (source machine)
`dev/fleet/fleet.sh package` -> `dist/depin-worker-fleet.zip` (~215MB). Delegates to
`dev/fleet/package-worker.sh`:
- control/coord defaults from `dev/fleet/worker-pkg/control.defaults.env` (git-ignored;
  tracked template `control.defaults.env.example`), flags override (`--coord`,
  `--control-host`, `--difficulty`, `--vc-port`, `--fs-port`, `--out`).
- `docker build --provenance=false --build-arg FLEET_* -t depin-worker:latest .`
  (separate tag; dev `depin-fleet:latest` + live control node untouched; single-arch so
  `docker load` works on older Docker).
- `docker save | gzip` + zip with `init.sh` / `.env.example` / `README.md`.

## Baking
Control/coord baked as image ENV via `Dockerfile.fleet` ARG/ENV (empty defaults ->
plain `fleet.sh build` unchanged): `FLEET_CONTROL_HOST FLEET_COORD_TRUST_SOURCE
FLEET_DIFFICULTY FLEET_VC_PORT FLEET_FS_PORT`. `entrypoint.sh` falls back to them for
`COORD_TRUST_SOURCE` + `VC_SERVER_ADDRESS`; **explicit runtime env always wins** so
existing single-host/control flows are unchanged.

## Target machine
unzip -> `cp .env.example .env` (set only `WORKER_COUNT` + a per-machine `START_INDEX`)
-> `./init.sh`. init.sh: require_docker -> `docker load` from tarball -> read baked
`FLEET_*` via `docker inspect .Config.Env` -> mint identities with **dockerized keytool**
(`docker run --entrypoint keytool --user $(id -u):$(id -g) -e HOME=/tmp`) -> gen
`docker-compose.workers.yml` -> preflight -> up. Subcmds: setup/status/logs/restart/down;
`SKIP_PREFLIGHT=1` bypasses the control-reachability gate.

## Gotcha fixed
Real coord trust source is `<nodeID>@tcp:<host>:7777` (has a `tcp:` scheme). Preflight
tcp-parse in **both** `fleet.sh` and init.sh needed `hostport="${hostport#tcp:}"`; the
original `fleet.sh workers-up` preflight was silently mis-parsing and failing the coord
check. See also depin-gitignore-allowlist-gotcha.md (had to allowlist the new tooling;
real control.defaults.env stays ignored like monitoring/.env).

## Scope
Workers only; control node (`worker-control.tunnel.appdemo.cyou`) + rollout stay on a
source machine. amd64-only. Coord trust source travels inside the zip (full-turnkey
choice) - it's a trust source, not a private key.
