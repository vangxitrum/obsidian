---
type: fact
tags: [aioz, depin, storage, hub, storj, go, coordinator, worker]
created: 2026-07-05
agent: main
---

**AIOZ DePIN "hub"** — core of the AIOZ DePIN decentralized storage network.
Located at `/home/tuan/work/depin-workspace/depin` (sibling repos: `storj`,
`uplink`, `storj`, `common`, `cookie` under `depin-workspace/`).

Distributed encrypted object storage over independent storage workers. Files
encrypted server-side by uplink, split with Reed–Solomon erasure coding,
distributed across workers so no single worker holds a full copy. Coordinator
tracks metadata, selects workers, issues signed access orders, audits network.

Strict layered peer architecture: Peer → Subsystem → Endpoint / Chore / Service →
Database. Full guide in `SOURCE_STRUCTURE.md`. Heavily modeled on **Storj** —
per `depin/CLAUDE.md` rule: keep components aligned with Storj (`../storj`) as
much as possible.

## Components (binaries under `cmd/`)
- **Coordinator** `cmd/coord` — network brain: metadata, worker selection, signed
  orders, audits, accounting, placement.
- **Worker** `cmd/worker` — storage node: stores encrypted pieces, serves
  up/downloads, reports health, runs GC.
- **Edge Server** `cmd/edgeserver` — gateway bridging external clients to network.
- **Uplink** `cmd/uplink` — client service/SDK for encrypted object up/download/manage.
- **Keytool** `cmd/keytool` — key/identity management (create keys, mnemonic recovery).
- **Version Control** `cmd/versioncontrol` — gates allowed binary versions.

## Coordinator split-scaling modes
- `api` — stateless gRPC endpoints, scales horizontally.
- `core` — background chores (singleton), runs periodic work exactly once.
- `run` — api + chores in one process (dev / small deploys).

## Build/proto
- Uses `buf` for protobuf (`buf.gen.yaml`, `buf.yaml`, `buf.lock`).
- Indexed by CodeGraph (`.codegraph/` present) — prefer `codegraph explore` over grep.
