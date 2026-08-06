---
type: fact
tags: [depin, fleet, worker-tags, auto-update, versioncontrol, docker]
created: 2026-08-05
agent: main
---

# Fleet host tag (`host=<machine>`) + fleet auto-update reality check

## What was added (2026-08-05, branch `feat/docker-bake-release`, uncommitted)

Fleet workers now advertise which MACHINE they run on, as a **self-signed** worker tag.
No `keytool sign-tags` round trip: `coord/contact` `processNodeTags` verifies against
`append(uc.nodeTagAuthority, self)`, so a worker's own signature is accepted for any tag
except `trusted_node=true` (that one still requires a real authority).

Worker-side support already existed (`worker/contact/tags.go` `GetTags`, config
`contact.self-signed-tags`, shipped by [[worker-tags-plan]]). Only fleet plumbing was
missing. New env, runtime-first with baked `FLEET_*` fallback:

- `HOST_TAG` - bare machine name, becomes tag `host=<value>`. Rejected if it contains
  `,` or `=`.
- `WORKER_TAGS` - extra `key=value,key=value`, appended. Each entry validated in the
  entrypoint, because the worker rejects the whole set (and fails to start) on a
  malformed pair, with a message that doesn't name the offender.

Both collapse into `AIOZ_WORKER_CONTACT_SELF_SIGNED_TAGS` (flag:
`--worker.contact.self-signed-tags`).

Files: `cmd/worker/entrypoint`, `dev/fleet/gen-fleet.sh`, `dev/fleet/fleet.sh` (docs +
`status` prints `host=`), `dev/fleet/Dockerfile.fleet`, `cmd/worker/Dockerfile`,
`dev/fleet/worker-pkg/init.sh`, `dev/fleet/worker-pkg/.env.example`.

In the portable package, `HOST_TAG` **defaults to `PREFIX`** - `PREFIX` already means
"which fleet is this", but it only separates fleets in Prometheus/Loki, which the
coordinator never sees. The host tag is the coordinator-side equivalent.

Read back: coord private debug `GET /worker-tags/?worker_id=<id>`
(`coord/contact/worker_tags_debug.go`).

## Traps found

1. **`fleet.sh package` / `package-worker.sh` only builds binaries when `bin/<name>` is
   MISSING** (`[ -x "$REPO/bin/$b" ] || missing=1`). An existing stale `bin/worker` is
   shipped silently. This produced a zip whose worker had no `self-signed-tags` flag at
   all while the source tree clearly had the feature. Always `make build-worker ...`
   explicitly before packaging. Symptom to recognise: `worker api --help | grep tag`
   empty, while a sibling non-user field like `check-in-timeout` is present.
2. `strings ./bin/worker | grep self-signed-tags` returns 0 even on a good binary -
   cfgstruct derives the kebab-case flag name from the Go field at runtime. Use
   `worker api --help`, not `strings`, to test for a config key.
3. The entrypoint's `cp -f /usr/local/bin/worker "$BIN"` is **unconditional**, so
   recreating any container to pick up the tag env also overwrites `/data/bin/worker`.
   On a node running an auto-updated binary, redeploying destroys that binary - which is
   exactly the evidence you may be trying to preserve.

## Fleet auto-update: inert everywhere as of 2026-08-05

Confirms and extends the incidental finding in [[component-version-tags]].

| fleet | `ENABLE_AUTO_UPDATE` | updater proc | outcome |
|---|---|---|---|
| local `fleet-worker-*` (10.0.0.67, 50) | unset -> false | absent | by design; `v0.0.1-116-gb82a3b6` (24 Jul) |
| `transcribe` `fleet-transcribe-worker-*` (10.0.0.106, 50) | **true** | running | fails every 60s |
| `oldwin` (10.0.0.202) | unknown | unknown | ssh `Permission denied (publickey)` |

Transcribe's config is correct (`--version.server-address http://10.0.0.67:10000`); the
**server does not exist**. Nothing listens on 10.0.0.67:10000, there is no
`fleet-versioncontrol` container, and the public control node
`worker-control.tunnel.appdemo.cyou` answers `/versions` with 404. So the loop logs
`connection refused` forever. Bring it back with `dev/fleet/fleet.sh control-up` on
10.0.0.67.

`v0.0.7` sightings are **historical, not live**: `dev/fleet/binaries/worker-v0.0.7.zip`
(31 Jul 11:59) is still `suggested` at `cursor: 100` in
`dev/fleet/versioncontrol.config.yaml`. Anything that polled during the window
versioncontrol was up on 31 Jul keeps that binary on its persistent `/data` volume
indefinitely. A worker reporting v0.0.7 proves auto-update worked *that day*, nothing more.

## Access gaps (blocked this investigation)

`oldwin` (both `github` and `transcribe` keys rejected), `demo` = coord 68.183.189.51,
`skeleton-worker`: publickey denied. `a-tunnel`/`b-tunnel`: server offers only `ssh-rsa`,
no matching host key type. `c-tunnel`: host key unverified. Only `transcribe` is reachable.

Local Prometheus carries **no worker `version` label** - every `version` series is
edgeserver. Do not try to answer "what version is each worker on" from it.

## Verification done

- `bash -n` on all 5 changed scripts.
- Go: `worker/contact` + `coord/contact` tag tests pass (sign -> verify -> store).
- Rebuilt `depin-worker:latest`, repackaged `dist/depin-worker-fleet.zip` (217M),
  unzipped it, ran `init.sh setup` against an unreachable coord: generated compose
  carried `HOST_TAG`/`WORKER_TAGS`, entrypoint logged
  `self-signed tags: host=tuan-desktop,site=hcm,rack=r1`, and that exact string was
  present in the running worker process's environ. Torn down after.
- Coordinator-side storage was NOT observed live (no reachable coord debug endpoint);
  it rests on the Go tests.

## Incidental

Root fs hit 100% mid-task and broke the Go linker (`ld: final link failed: No space left
on device`). Freed ~44GB by removing 55 tagged images with no container (all `*depin*`,
`coord:*`, `edgeserver:*`, `aioz-stream*` deliberately excluded; non-forced `docker rmi`
as a second safety net). The 45 dangling anonymous volumes (~21GB) were left alone.
Prometheus was already failing writes with `no space left on device` before this.
