# uplink-sdk → worker upload (full segment cycle)

Wire the SDK's `UploadFile` end-to-end against a real worker: encrypt + erasure-encode
each segment, sign per-piece Orders and PieceHashes with a piece key, stream the
pieces to workers via the worker's `PieceStoreService` over TCP gRPC, and close
the loop so coord's `CommitFile` marks the file `READY`.

## Goals

- A single `client.UploadFile(ctx, reader, params)` call uploads a small file end-to-end
  against a locally-running worker and lands the file in coord's `files` table with
  status `READY`.
- Signing is real: SDK signs `Order` and `PieceHash` with a `vo.PiecePrivateKey`;
  worker verifies them via the existing `pkg/signing` helpers.
- Worker reports each successful piece back to coord so the `CommitFile` confirmed
  count is satisfied.

## Non-goals

- TLS / mTLS between SDK and worker (TCP-insecure for now).
- Connection pooling across pieces; per-piece dial is fine.
- Retry queues / async re-confirm for failed worker→coord reports.
- Multi-worker placement strategy — the dev single-worker placement (id=999) stays
  the target.
- Download path.

## Architecture overview

```
SDK ──CreateFile──► coord
SDK ──BeginSegment─► coord  ─► returns []shared.OrderLimit (one per worker)
   for each OrderLimit (concurrent):
     SDK ──Upload(stream)──► worker (piece.v1.PieceStoreService)
                              worker stores piece, returns UploadResponse
                              worker ──ReportPieceUploaded──► coord  [piece row → WorkerConfirmed]
   SDK ──CommitSegment──► coord  [piece rows → UplinkCommitted]
SDK ──CommitFile──► coord  [counts WorkerConfirmed; if every segment has ≥ required, file → READY]
```

## Progress

- [x] Section 1 — Worker bring-up
- [ ] Section 2 — SDK piece-signing key
- [ ] Section 3 — SDK workerclient
- [ ] Section 4 — Worker → coord report
- [ ] Section 5 — Tests

## Section 1 — Worker bring-up ✅

Pre-flight; without a running worker checked in to coord, `BeginSegment` returns no
order limits.

- Reuse the worker identity that `gen_identities_dev.sh` already produces at
  `identities/worker-1/identity.{cert,key}`.
- Worker uses its own SQLite at `Storage.DatabaseDir`; no postgres dependency.
- `dev/worker1/config.yaml` generated via `make setup-worker`. Key fields:
  - listen address `:28967` (TCP)
  - private address `127.0.0.1:28968`
  - `Storage.DatabaseDir` at `dev/worker1/data/`
- `Makefile.build` targets:
  - `setup-worker` — runs setup with `--config-dir ./dev/worker1 --identity-dir ./identities/worker-1`.
  - `build-worker` — mirrors `build-coord` (host `GOOS/GOARCH`, `-tags purego`, `CGO_ENABLED=1`).
  - `run-worker` — runs `api` subcommand with the dev config.
- Verify: after `make run-worker`, worker checks in to coord.

## Section 2 — SDK piece-signing key

The worker's `PieceStoreService.Upload` verifies two uplink signatures (see
`worker/application/usecase/piecestore.go:320,343`):

- `Order.UplinkSignature` over the `OrderSigning` bytes, key = `Order.UplinkPublicKey`.
- `PieceHash.Signature` over the `PieceHashSigning` bytes, key = `OrderLimit.UplinkPublicKey`.

The SDK therefore needs a `vo.PiecePrivateKey`. It is **separate** from:

- The TLS identity key (`identity.key`) — used for mTLS to coord.
- `priv_key.json` — cosmos signer used by `RegisterAccount`.

### Key generation

- New keytool subcommand under coord's keytool group:
  `go run ./cmd/coord keytool new-piece --out <path>` produces a JSON file:
  ```json
  { "private_key": "<hex bytes>" }
  ```
  using `vo.NewPiecePrivKeyFromBytes` / a small marshaller.
- New `Makefile.build` target: `setup-uplink-piece-key` calls the keytool to write
  `identities/uplink/piece_key.json`.

### SDK surface

- New option: `WithPieceKey(path string)` on `uplink-sdk.Client`.
- `New(...)` also auto-loads `<identityDir>/piece_key.json` if present (mirrors the
  existing `loadSignerIfPresent` for `priv_key.json`). Explicit option wins.
- `UploadFile` returns an error if no piece key is available:
  `"uplinksdk: upload requires WithPieceKey or <identityDir>/piece_key.json"`.
  Methods that don't sign anything (`RegisterAccount`, `ListPlacements`,
  `CreateContract`, `CreateFile`) are unaffected.

## Section 3 — SDK workerclient

Replace the broken `pkg/workerclient` (it imports `piecestore/v1` which the worker
does not serve) with a new SDK-private client that uses `piece.v1.PieceStoreService`.

### Location

`uplink-sdk/internal/workerclient/`. Leave `pkg/workerclient` alone; its only known
consumer is `uplink/`, which is not part of this work. Clean up in a separate task
once nothing imports it.

### API

```go
package workerclient

import (
    "context"
    "io"

    "aioz-depin/internal/vo"
    piecepb "aioz-depin/pkg/pb/worker/piece/v1"
    sharedpb "aioz-depin/pkg/pb/shared"

    "go.uber.org/zap"
)

type Client struct {
    pieceKey *vo.PiecePrivateKey
    logger   *zap.Logger
}

func New(pieceKey *vo.PiecePrivateKey, logger *zap.Logger) *Client

// PutPiece dials coordLimit.WorkerAddress over insecure TCP gRPC, opens the
// Upload bidi stream, signs an Order + the final PieceHash with the piece
// key, streams the piece bytes, and returns the worker's PieceHash response.
func (c *Client) PutPiece(
    ctx context.Context,
    coordLimit sharedpb.OrderLimit,
    data io.Reader,
) (*piecepb.PieceHash, error)
```

### Per-piece flow inside `PutPiece`

1. Build `piecepb.OrderLimit` from `coordLimit`:
   - copy `PieceId`, `WorkerId`, `SerialNumber`, `Limit`, `Action`, `OrderCreation`,
     `OrderExpiration`, `PieceExpiration`, `CoordinatorId`, `CoordinatorSignature`.
   - set `UplinkPublicKey = c.pieceKey.PublicKey()`.
2. Dial `coordLimit.WorkerAddress` via `grpc.Dial(addr, grpc.WithInsecure())`.
   Per-piece dial; close on return.
3. Open the bidi `Upload` stream.
4. **Frame 1** — handshake: `{ Limit, HashAlgorithm: SHA256, Action: PUT }`.
5. **Sign Order** — `Order{ SerialNumber, Amount: coordLimit.Limit, UplinkPublicKey }`
   passed to `signing.SignUplinkOrder` (with the piece key). Send as **Frame 2**.
6. **Stream chunks** — fixed 2 KiB buffer (matches the existing copy). Tee through
   a SHA-256 hasher to compute the running hash for the final frame.
7. **Final frame** — `Done` = a `PieceHash{ PieceId, Hash: hasher.Sum, PieceSize,
Timestamp: coordLimit.OrderCreation, HashAlgorithm: SHA256 }`, signed via
   `signing.SignUplinkPieceHash`.
   Implementation note: `SignUplinkOrder` takes `vo.PiecePrivateKey`,
   `SignUplinkPieceHash` takes `crypto.PrivateKey`. The piece-key loader
   must expose both forms (the underlying ed25519 key fits both).
8. `CloseAndRecv` returns `UploadResponse{ PieceId }`. Map to `piecepb.PieceHash`
   and return.

Errors during steps 4–8 cancel the stream and propagate.

### Per-segment fan-out

The existing `upload.go:155 uploadSegment` already fans out one goroutine per
order limit, cancels remaining once `OptimalThreshold` is reached, and fails the
segment if `RequiredCount` is not met. Keep that wrapper unchanged — just swap
the per-piece call from `pkg/workerclient` to the new `uplink-sdk/internal/workerclient`.

## Section 4 — Worker → coord report

`CommitFile` in coord counts pieces in status `PieceUploadWorkerConfirmed`
(`coord/infrastructure/repository/segment_piece_upload.go:54`). Nothing currently
transitions rows from `UplinkCommitted` → `WorkerConfirmed`. We close the loop
by having the worker call coord's `FileService.ReportPieceUploaded` after every
successful piece store.

### Where the change goes

`worker/application/usecase/piecestore.go`. After the existing `Store(...)` success
path produces the worker's own signed `PieceHash`, add:

```go
conn, err := s.coordDialer.DialNode(ctx, s.coordPeerURL, dial.DialOptions{})
if err != nil {
    s.log.Warn("report piece upload: dial coord", zap.Error(err))
    return // piece is on disk; coord can reconcile later
}
defer conn.Close()
if _, err := filepb.NewFileServiceClient(conn).ReportPieceUploaded(ctx,
    &filepb.ReportPieceUploadedRequest{ Limit: *orderLimit, PieceHash: *workerHash },
); err != nil {
    s.log.Warn("report piece upload: rpc", zap.Error(err))
}
```

### Wiring

- `PieceStoreService` struct grows two fields: `coordDialer dial.Dialer` and a
  way to resolve coord's PeerURL.
- **Coord discovery uses the trust list** (already populated by the contact loop).
  Read coord's id from worker config (it's already there for `dialCoord`) and
  call `s.trust.GetNodeURL(ctx, coordID)` to get a `vo.PeerURL`. Same path the
  contact CheckIn uses (`worker/application/usecase/contact.go:198`).
- DI: `worker/core.go` already constructs `Dialer` and `trust.Service`; pass them
  to `NewPieceStoreService` at construction.

### Failure mode

Report failure is logged + ignored. The piece is on disk, the worker isn't blocked,
and the SDK will see `CommitFile` fail with `InvalidStatus` if too few report
calls succeeded. For dev, rerunning fixes it. A real retry path is future work.

## Section 5 — Tests

New integration tests in `uplink-sdk/test/integration_test.go`. Same env-var
gating + `-run` targetability as the existing ones.

- `TestBeginSegment` —
  `CreateContract` → `CreateFile` → `BeginSegment(fileID, 1)`. Assert
  `OrderLimits` is non-empty, every limit has a `WorkerAddress` and a
  `CoordinatorSignature`. Read-only otherwise; fast.
- `TestUploadSegment` —
  ~5 MiB random payload, `client.UploadFile`. Asserts a non-zero `FileID`. Then
  a direct DB read to confirm `files.status` for that id is `READY`.

Helper:

- `skipIfNoWorkers(t, db)` — `SELECT count(*) FROM workers`; `t.Skip` if zero.
  Called at the top of each test. Avoids confusing failures when the user hasn't
  run `make run-worker`.

## Implementation order

1. ~~**Worker bring-up.**~~ ✅ `dev/worker1/config.yaml`, Make targets, build verified.
2. **Piece key.** Keytool subcommand, generator, `WithPieceKey` option + auto-load,
   `setup-uplink-piece-key` target.
3. **SDK workerclient.** `uplink-sdk/internal/workerclient/` with the new
   `Client.PutPiece`; wire it into `uplink-sdk/upload.go uploadSegment`.
4. **Worker → coord report.** `PieceStoreService` gets the dialer + trust, calls
   `FileService.ReportPieceUploaded` post-store.
5. **Tests.** `TestBeginSegment`, `TestUploadSegment`, `skipIfNoWorkers` helper.

## Open decisions (locked in this brainstorm)

- TCP for the worker for now (no TLS).
- Per-piece dial; no pooling.
- Hash algorithm: SHA-256.
- Coord discovery in the worker: trust list, not config or proto change.
- Full PiecePrivateKey signing (not the dev-disabled-signature path).
