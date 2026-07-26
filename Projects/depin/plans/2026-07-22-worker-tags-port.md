---
type: plan
project: depin
created: 2026-07-22
tags: [depin, worker-tags, node-tags, plan]
---

# Plan: Port Storj Node Tags → depin "Worker Tags"

## Context

`storj/worker-tags.md` documents Storj's **node tags**: cryptographically signed
`key=value` pairs a storage node pushes to the satellite on check-in, verified
against a trust authority, stored in the overlay DB, and consumed by node
selection/placement (geofencing, reserved pools, datacenter/rack topology).

depin mirrors Storj (`worker`=storagenode, `coord`=satellite) but has **no
node-tag concept today** (confirmed: repo-wide search for `nodetag`/`NodeTagSet`/
`UpdateNodeTags` returns nothing; the existing `coord/tags` package is unrelated
S3 object/bucket tagging). This ports the feature so operators can attach signed
attributes to a worker.

**Scope decided with the user:**
- **Foundation first** — worker advertises signed tags → coord verifies + stores
  them → tags are **readable** (coord debug endpoint). The
  placement/selection *consumption* is a documented **Phase 2 follow-up**, not
  built now (keeps the selection hot path untouched in v1).
- **Full signing parity** — both **self-signed** (worker signs its own tags) and
  **authority-signed** tags (a trusted cert signs tags for a specific worker),
  including the special `trusted_node`→`worker.is_trusted` semantics, a coord
  `TagAuthorities` trusted-cert config, and a `keytool` command to mint
  authority-signed blobs.

depin's `pkg/signing`, `pkg/identity`, `pkg/pkcrypto` are near-verbatim ports of
`storj.io/common`, so the Storj source maps almost 1:1.

Storj → depin naming: `storagenode`→`worker`, `satellite`→`coord`,
`storj.NodeID`→`internal/vo.UUID`, `storj.io/common/signing`→`pkg/signing`,
`storj.io/common/identity`→`pkg/identity`, `shared/nodetag`→`pkg/nodetag`.

---

## Task 1 — Proto messages

Port the nodetag messages from `common/pb/nodetags.proto`.

- **New** `pkg/pb/shared/nodetag/v1/nodetag.proto` (package `shared.nodetag.v1`,
  `option go_package = "aioz-depin/pkg/pb"`). Messages, kept byte-stable so
  signatures verify:
  - `Tag { string name = 1; bytes value = 2; }`
  - `NodeTagSet { bytes node_id = 1; repeated Tag tags = 2; int64 signed_at = 3; }`
  - `SignedNodeTagSet { bytes serialized_tag = 1; bytes signer_node_id = 3; bytes signature = 4; }`
  - `SignedNodeTagSets { repeated SignedNodeTagSet tags = 1; }`
  - Keep `node_id`/`signer_node_id` as raw `bytes` (parsed via `vo.UUID` in Go) —
    do **not** use gogoproto customtype, so the marshaled `serialized_tag` stays
    stable across signer/verifier.
- **Edit** `pkg/pb/coord/contact/v1/contact.proto`: import the new proto and add
  to `CheckInRequest`: `shared.nodetag.v1.SignedNodeTagSets signed_tags = 7;`
  (nullable).
- Regenerate: `make proto` (→ `buf generate`, per `Makefile.build:252`). Confirm
  generated Go appears and `go build ./pkg/pb/...` passes.

## Task 2 — `pkg/nodetag` signing package (new)

Verbatim port of `storj/shared/nodetag/{sign.go,authority.go}` onto depin
primitives. New package `pkg/nodetag` (depin has no `shared/` dir; `pkg/` is the
home for `signing`/`identity`).

- `pkg/nodetag/sign.go`:
  - `Sign(ctx, *nodetagpb.NodeTagSet, signing.Signer) (*nodetagpb.SignedNodeTagSet, error)`
    — `pb.Marshal` the set, `signer.HashAndSign`, fill signature + `signer_node_id`
    (`signer.ID().Bytes()`) + `serialized_tag`.
  - `Verify(ctx, *SignedNodeTagSet, signing.Signee) (*NodeTagSet, error)` — check
    `signer_node_id == signee.ID()`, `HashAndVerifySignature`, unmarshal.
  - Error classes `SignatureErr`, `SerializationErr`, `WrongSignee` (use
    `github.com/zeebo/errs`, already a dep).
- `pkg/nodetag/authority.go`:
  - `type Authority []signing.Signee`; `Verify` (iterate, match signer id);
    `Include(id vo.UUID) bool`; `UnknownSignee` error.
  - `LoadAuthorities(peerIdentity *identity.PeerIdentity, locations string) (Authority, error)`
    — seed with `signing.SigneeFromPeerIdentity(peerIdentity)` then, for each
    comma-separated PEM path, `identity.PeerIdentityFromPEM` +
    `signing.SigneeFromPeerIdentity`.
- **Reuse:** `pkg/signing` (`Signer`/`Signee`, `SignerFromFullIdentity`,
  `SigneeFromPeerIdentity` — `pkg/signing/peers.go`), `pkg/identity`
  (`PeerIdentityFromPEM`), `pkg/pb`.
- Port `sign_test.go` (round-trip sign/verify + authority match/`UnknownSignee`).

## Task 3 — Worker: assemble + advertise tags

Port `storj/storagenode/contact/tags.go` (`GetTags`) and thread the signer in.

- **New** `worker/contact/tags.go`: `GetTags(ctx, cfg Config, id *identity.FullIdentity) (*nodetagpb.SignedNodeTagSets, error)`:
  - base64-decode `cfg.Tags` → `SignedNodeTagSets` (authority-signed, pre-minted).
  - if `cfg.SelfSignedTags` non-empty: build `NodeTagSet{NodeId: id.ID.Bytes(),
    SignedAt: time.Now().Unix()}`, `strings.Cut` each `key=value`, append `Tag`s,
    `nodetag.Sign(ctx, set, signing.SignerFromFullIdentity(id))`, append to the set.
- **Edit** `worker/contact/service.go`:
  - `Config` (lines 37-43): add `Tags string` (help: "base64-encoded
    authority-signed SignedNodeTagSets blob", default `""`) and
    `SelfSignedTags string` (help: "comma-separated key=value tags, self-signed
    by this worker", default `""`). (Use `string`+split, not `[]string`, to stay
    within the confirmed cfgstruct field-tag styles at
    `internal/cfgstruct/cfgstruct.go`.)
  - `Service` struct (46-63) + `NewService` (74-87): add `identity
    *identity.FullIdentity` and `config Config` (or just the two tag strings).
  - `pingCoordOnce` (build site 171-179): call `GetTags` and set
    `SignedTags:` on the `CheckInRequest`.
- **Edit** `worker/peer.go:350-358`: pass `peer.Identity` and `config.Contact`
  into `contact.NewService`. (`WorkerInfo.ID = peer.ID() = peer.Identity.ID`, so
  self-signed `node_id` matches the check-in `id` — verified.)

## Task 4 — Coord: verify + store (the `processNodeTags` analog)

Port `storj/satellite/contact/service.go:183 processNodeTags` + `verifyTags`.

- **New model** in `coord/contact/` (mirrors `nodeselection.NodeTag`):
  `type WorkerTag struct { WorkerID vo.UUID; Name string; Value []byte; SignedAt
  time.Time; Signer vo.UUID }`, `type WorkerTags []WorkerTag` with
  `FindBySignerAndName`.
- **Edit** `coord/contact/service.go`:
  - `Config` (19-50): add `TagAuthorities string` (help: "comma-separated PEM cert
    paths trusted to sign worker tags").
  - `NewContactUsecase` (65-): add param `nodeTagAuthority nodetag.Authority`,
    store on the usecase.
  - `CheckIn` (155-269): add params `signedTags *nodetagpb.SignedNodeTagSets, self
    signing.Signee`. Add `processNodeTags`: for each signed tag, `verifyTags`
    against `append(uc.nodeTagAuthority, self)`, require `NodeTagSet.node_id ==
    worker.Id`; if `name=="trusted_node" && value=="true" &&
    uc.nodeTagAuthority.Include(signer)` set `worker.IsTrusted=true`; else collect
    a `WorkerTag`. Before/at the existing `store.AddWorker` call (264), also call
    `store.UpdateWorkerTags(ctx, tags)` when non-empty. Bad signatures are
    logged + skipped (never fail the check-in) — same as Storj.
- **Edit** `coord/contact/endpoint.go:57-130`: build `self :=
  signing.SigneeFromPeerIdentity(peerIden)` and pass `req.SignedTags` + `self`
  into `contactUsecase.CheckIn`.
- **Edit** `coord/contact/worker.go` (Worker struct, 10-55): add `IsTrusted bool`
  (column `is_trusted`); it rides the existing `AddWorker` upsert.
- **Edit** `coord/contact/store.go` (Store interface): add
  `UpdateWorkerTags(ctx, WorkerTags) error` and `GetWorkerTags(ctx, vo.UUID)
  (WorkerTags, error)`.
- **Edit** `coord/peer.go` `setupContact` (285-301): build the authority via
  `nodetag.LoadAuthorities(b.Identity.PeerIdentity(), b.Config.Contact.TagAuthorities)`
  and pass into `NewContactUsecase`.

## Task 5 — Coord DB: `worker_tags` table + repo

Faithful port of Storj's signed `node_tags` table (keyed by node+name+signer,
retains signer & signed_at — supports multiple authorities per worker). Not a
JSONB column.

- **New migration** `coord/db/migrations/000033_worker_tags.up.sql` (next number;
  golang-migrate, auto-embedded via `coord/db/embed.go` glob, run by
  `coord/db/database.go:56-88`):
  - `CREATE TABLE IF NOT EXISTS worker_tags ( worker_id TEXT, name TEXT, value
    BYTEA, signed_at TIMESTAMPTZ, signer TEXT, PRIMARY KEY (worker_id, name,
    signer) )` + index on `worker_id`.
  - `ALTER TABLE workers ADD COLUMN IF NOT EXISTS is_trusted BOOLEAN NOT NULL
    DEFAULT false`.
  - matching `.down.sql`.
- **New** `coord/db/worker_tags_repo.go` (gorm, alongside `worker_repo.go`):
  implement `UpdateWorkerTags` with upsert/**replace** semantics
  (`clause.OnConflict{Columns: worker_id,name,signer, DoUpdates: value,signed_at}`
  — mirrors Storj's `create ( replace )`) and `GetWorkerTags`. Wire into
  `WorkerRepository` (the `contact.Store` impl) so `setupContact` keeps one store.
  Note: `workers` schema is SQL-migration-owned (AutoMigrate off,
  `worker_repo.go:27-33`) — the new column/table must be in the migration.
- Add `IsTrusted` to the worker column list in `AddWorker`'s `db.Save` path if the
  repo enumerates columns.

## Task 6 — Keytool: mint authority-signed tags

- **New** `cmd/keytool/cmd_sign_tags.go` (beside `cmd_sign.go`): subcommand
  `keytool sign-tags --identity-dir <authority> --node-id <worker-uuid> --tags
  key=value,key2=value2` → load authority `FullIdentity`
  (`identity.Config.Load`), build `NodeTagSet{NodeId: workerUUID.Bytes(),
  SignedAt: now, Tags:...}`, `nodetag.Sign` with
  `signing.SignerFromFullIdentity`, wrap in `SignedNodeTagSets`, `pb.Marshal` +
  base64, print. Operator pastes output into the worker's
  `--worker.contact.tags`. The authority's cert must be listed in the coord's
  `TagAuthorities` (or be the coord's own identity, which `LoadAuthorities`
  trusts automatically).
- (Optional, note only) A `TagSigner` gRPC service (Storj) is a Phase-2 nicety;
  the offline CLI is sufficient for the foundation.

## Task 7 — Readable: coord debug endpoint

Make stored tags observable (since placement consumption is deferred).

- **New** `coord/contact/worker_tags_debug.go`: a `debug.Extension`
  (`Description`/`Path()="/worker-tags/"`/`Handler`) that reads
  `?worker_id=<uuid>` via `store.GetWorkerTags` and returns JSON (name,
  value as utf8+hex, signer, signed_at) plus the worker's `is_trusted`.
- Register it on the coord's `debug.Server` where it's constructed (grep
  `debug.NewServer`/extensions in coord startup; `debug.Server` accepts variadic
  `Extension`s — `pkg/debug/server.go:52-70,150-152`). Mirrors the existing
  extension-endpoint pattern.

---

## Deferred — Phase 2 (documented, NOT in this plan)

Placement/selection consumption, to be a follow-up once the foundation lands:
- Add `Tags WorkerTags` (+ `IsTrusted`) to the `SelectedWorker` projection
  (`coord/contact/selectedworker.go`) and populate it by joining `worker_tags`
  in `participatingRow`/`projectWorker` (`coord/db/worker_repo.go:193-386`).
- Add a typed `TagFilter` to `coord/placement/worker_filters.go` (implements
  `WorkerFilter.Match`) + its `SerializeFilter`/`DeserializeFilter` case in
  `coord/placement/filter_wrapper.go` (type `"tag"`).
- Feed it through the selection caches (`coord/overlay/uploadselection.go`,
  `downloadselection.go`).
- Optionally gate selection/hashstore behavior on `worker.is_trusted`.

---

## Verification

**Unit (`go test ./...`, no cluster needed):**
- `pkg/nodetag`: sign/verify round-trip; authority match; `UnknownSignee`;
  tampered `serialized_tag`/`signature` rejected; wrong-signee rejected (ported
  from `sign_test.go`).
- `worker/contact`: `GetTags` — self-signed only, authority-blob only, both
  merged; `node_id == identity.ID`.
- `coord/contact`: `processNodeTags` — valid self-signed stored; valid
  authority-signed stored; bad signature dropped (check-in still succeeds);
  `node_id` mismatch rejected; `trusted_node=true` sets `is_trusted` **only** for
  an authority signer, not a self-signer.

**DB (needs Postgres):** `worker_tags_repo` upsert/replace idempotency (re-check-in
same tag updates in place; different signer adds a row) + `GetWorkerTags`.

**Build/gen:** `make proto` regenerates cleanly; `go build ./...` passes.

**End-to-end (dev cluster / testplanet if available):**
1. `keytool sign-tags` for a worker with `trusted_node=true,dc=ewr1` using an
   authority identity listed in coord `--<coord>.contact.tag-authorities`.
2. Start worker with `--worker.contact.tags=<blob>` and
   `--worker.contact.self-signed-tags=rack=42`.
3. After a check-in, `GET http://<coord-debug>/worker-tags/?worker_id=<uuid>`
   shows `dc=ewr1` + `rack=42`, and `is_trusted=true`.
4. Corrupt one byte of the authority blob → that tag is dropped, check-in still
   succeeds, `is_trusted` stays false; a self-signed `trusted_node=true` does
   **not** grant trust.

## Key files (touch list)
- proto: `pkg/pb/shared/nodetag/v1/nodetag.proto` (new),
  `pkg/pb/coord/contact/v1/contact.proto`
- signing: `pkg/nodetag/{sign,authority,sign_test}.go` (new)
- worker: `worker/contact/tags.go` (new), `worker/contact/service.go`,
  `worker/peer.go:350-358`
- coord: `coord/contact/{service,endpoint,worker,store}.go`,
  `coord/contact/worker_tags_debug.go` (new), `coord/peer.go` `setupContact`
- db: `coord/db/migrations/000033_worker_tags.{up,down}.sql` (new),
  `coord/db/worker_tags_repo.go` (new)
- keytool: `cmd/keytool/cmd_sign_tags.go` (new)
