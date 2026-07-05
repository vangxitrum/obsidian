# Component README Checklist

Tick `[x]` the components that should get a `README.md` in their folder.
Each line shows the **target README path** for that component.

> How to use: change `[ ]` → `[x]` for every component you want documented, then tell me to generate the checked ones.

> **Status (2026-06-23):** all 41 checked components have been generated (rich,
> hashstore-style READMEs; cross-cutting topics in `docs/`). Verified: 41/41 files
> present, 382/382 relative links resolve, key symbols spot-checked against source.
> Not generated (left unchecked): `coord/wallet`, uplink client (group 4), uplink-SDK
> (group 5), `pkg/pb`, `internal/grpcutil`, `internal/mud`, `internal/compensation`.

---

## 1. Binaries / Peers (entrypoints)

- [x ] **coord** — Coordinator peer (API / Core / Audit roles) → `coord/README.md`
- [x] **worker** — Storage node peer → `worker/README.md`
- [x] **edgeserver** — S3/HTTP gateway → `edgeserver/README.md`
- [x] **uplink (CLI)** — Client command-line tool → `cmd/uplink/README.md`
- [x] **versioncontrol** — Version/rollout control server → `versioncontrol/README.md`

## 2. Coordinator subsystems (`coord/`)

- [x] **overlay** — Worker registry & node selection → `coord/overlay/README.md`
- [x] **contact** — Worker check-in / liveness → `coord/contact/README.md`
- [x] **orders** — Bandwidth order signing & settlement → `coord/orders/README.md`
- [x] **audit** — Proof-of-storage verification → `coord/audit/README.md`
- [x] **accounting** — Storage/bandwidth tally → `coord/accounting/README.md`
- [x] **placement** — Geo/placement constraint engine → `coord/placement/README.md`
- [x] **file / expireddeletion** — Object lifecycle & expiry GC → `coord/file/README.md`
- [ ] **wallet** — Payment/wallet integration → `coord/wallet/README.md`
- [x] **geoip** — Worker geo-location → `coord/geoip/README.md`
- [x] **storage** — Metadata storage layer → `coord/storage/README.md`
- [x] **server / healthcheck** — HTTP server + health → `coord/server/README.md`
- [x] **db** — Persistence (migrations, seeds, coorddbtest) → `coord/db/README.md`

## 3. Worker subsystems (`worker/`)

- [x] **piecestore** — Piece upload/download endpoint (hot path) → `worker/piecestore/README.md`
- [x] **hashstore** — Hash-based blob backend → `worker/pkg/hashstore/README.md`
- [x] **blobstore** — Filesystem blob backend → `worker/pkg/blobstore/README.md`
- [x] **filewalker** — Disk scanning for GC/audit → `worker/pkg/filewalker/README.md`
- [x] **retain** — Bloom-filter garbage collection → `worker/retain/README.md`
- [x] **orders** — Order persistence & settlement → `worker/orders/README.md`
- [x] **contact** — Check-in with coordinator → `worker/contact/README.md`
- [x] **monitor** — Disk space / health monitoring → `worker/monitor/README.md`
- [x] **trust** — Trusted coordinator list → `worker/trust/README.md`
- [x] **db** — Persistence (migrations, dbtest) → `worker/db/README.md`

## 4. Uplink client (`uplink/` — Clean Architecture)

- [ ] **application/service** — Upload/download orchestration → `uplink/application/service/README.md`
- [ ] **domain** — Core domain model (entity/repo/services) → `uplink/domain/README.md`
- [ ] **delivery/http** — HTTP API + middleware → `uplink/delivery/http/README.md`
- [ ] **infras** — Persistence (db/sqlite) → `uplink/infras/README.md`
- [ ] **internal/eestream** — Erasure coding → `uplink/internal/eestream/README.md`
- [ ] **internal/keymanager** — Key management → `uplink/internal/keymanager/README.md`
- [ ] **pkg/streams** — Stream handling → `uplink/pkg/streams/README.md`

## 5. Uplink SDK (`uplink-sdk/` — data-plane pipeline)

- [ ] **pieceupload / piecedownload** — Per-piece transfer → `uplink-sdk/internal/pieceupload/README.md`
- [ ] **segmentupload / segmentdownload** — Per-segment transfer → `uplink-sdk/internal/segmentupload/README.md`
- [ ] **splitter** — Stream splitting → `uplink-sdk/internal/splitter/README.md`
- [ ] **streambatcher** — Stream batching → `uplink-sdk/internal/streambatcher/README.md`
- [ ] **scheduler** — Concurrency scheduler → `uplink-sdk/internal/scheduler/README.md`
- [ ] **workerclient** — Worker RPC client → `uplink-sdk/internal/workerclient/README.md`

## 6. Shared libraries (`pkg/`)

- [ ] **pb** — Protocol Buffers / gRPC contracts → `pkg/pb/README.md`
- [x] **eestream + infectious** — Reed-Solomon erasure coding → `pkg/eestream/README.md`
- [x] **encryption / encoding** — Crypto primitives → `pkg/encryption/README.md`
- [x] **merkle** — Merkle trees → `pkg/merkle/README.md`
- [x] **bloomfilter** — Bloom filters → `pkg/bloomfilter/README.md`
- [x] **identity** — Node identity / PKI → `pkg/identity/README.md`
- [x] **signing / pksigning / pkcrypto** — Signature & crypto → `pkg/signing/README.md`
- [x] **modular + config** — DI framework → `pkg/modular/README.md`
- [x] **lifecycle** — Component lifecycle → `pkg/lifecycle/README.md`
- [x] **version** — Version/build info → `pkg/version/README.md`

## 7. Internal infrastructure (`internal/`)

- [x] **testplanet** — In-process integration harness → `internal/testplanet/README.md`
- [ ] **grpcutil** — Custom gRPC transport (incl. P2P) → `internal/grpcutil/README.md`
- [ ] **mud** — DI container → `internal/mud/README.md`
- [ ] **compensation** — Worker payment calculation → `internal/compensation/README.md`

## 8. Cross-cutting docs (not folder-bound — go in `docs/`)

- [x] **Identity & trust model** → `docs/identity-trust.md`
- [x] **Data flow: upload path** → `docs/dataflow-upload.md`
- [x] **Data flow: audit path** → `docs/dataflow-audit.md`
- [x] **gRPC / P2P transport** → `docs/transport.md`
- [x] **Configuration system** → `docs/configuration.md`
