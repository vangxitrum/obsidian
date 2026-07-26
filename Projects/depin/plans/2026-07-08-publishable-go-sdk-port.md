---
type: plan
project: depin
created: 2026-07-08
---

# Plan: Publishable standalone `go-sdk` (ported & de-coupled from depin)

## Context

`depin/uplink-sdk` is a complete, working Go client for the coord service
(register / upload / download), but it lives inside the `aioz-depin` monorepo and
is welded to it through `internal/*` packages — and, critically, drags in the
**private forks** at `10.0.0.50/aioz-network/*` (cosmos-sdk, ethermint, ibc-go,
go-aioz, Gravity-Bridge, bech32-ibc) via `internal/vo`.

The `go-sdk` repo (`gitlab.internal:aioz-depin/go-sdk`, currently empty) must
become a **publishable** standalone Go module. Confirmed requirements:

1. **Standalone module** — copy the dependency closure in; no `aioz-depin`
   imports remain.
2. **Publishable — ZERO private deps.** No `github.com/cosmos/cosmos-sdk`,
   `evmos/ethermint`, `ibc-go`, or any `10.0.0.50/aioz-network/*` replace. Only
   public modules (libp2p, grpc, gogo/protobuf, go-ethereum, btcec, zeebo/errs,
   monkit, google/uuid, zap) are allowed.
3. **Full API surface** — register + upload + download + tickets + placements +
   contracts + low-level segment entries.
4. **Remove the multi-worker piece fan-out** — sequential piece transfer.
5. **Add debug logging** throughout upload/download for step-by-step tracing.
6. **Reference `storj.io/uplink` (`../uplink`)** for API ergonomics, package
   layout, and tracing style (it is the upstream the aioz SDK was modeled on).

The SDK already implements all three flows (README's "download is a stub" is
stale): `RegisterAccount`/`GetAccount` (`account.go`), `UploadFile`/`CreateFile`
(`upload.go`), `DownloadFile`/`DownloadByTicket`/`DownloadManifest`
(`download.go`). This is a **packaging + de-coupling** job, not new features.

## The publishability blocker and how narrow it is

Full-closure grep confirms the private-fork coupling is isolated to **exactly
three value types** in `internal/vo`, all reachable from one SDK site
(`RegisterAccount` → `Client.public_key` / `Client.address`):

| Type | File | Cosmos use | Wire risk |
|---|---|---|---|
| `vo.PrivKey` | `privkey.go` | cosmos `Any` + ethermint privkey | Low — reproducible from public secp256k1 |
| `vo.PublicKey` | `publickey.go` | cosmos `Any`, gogo customtype on `RegisterRequest`/`Client` | **HIGH — must byte-match** ethermint `Any`: `type_url="/ethermint.crypto.v1.ethsecp256k1.PubKey"`, value `0x0a 0x21 ‖ <33-byte compressed pubkey>` |
| `vo.AccAddress` | `address.go` | `sdk.AccAddress` bech32/EVM | Low — 20-byte blob; hex via go-ethereum |

Everything else is already public-safe:
- `vo.UUID`, `vo.StreamID`, `vo.PieceID`, `vo.SerialNumber`, `vo.PeerURL`,
  `vo.PiecePrivateKey`/`PiecePublicKey` (**ed25519**) — no cosmos.
- `pkg/identity` mTLS load path (`Config.Load` → `pkcrypto` → stdlib
  `crypto/x509`+`ed25519`) — no cosmos. Imports only `vo` (for `vo.UUID`) and
  `pkcrypto`.
- All file / storage / piece / order protobuf messages the SDK sends/receives
  use only cosmos-free customtypes.

Two extra cosmos-coupled files exist in the closure but are **not on the SDK
path** and are simply dropped:
- `pkg/identity/key_manager.go` (`KeyLibp2p`) — unused by the SDK.
- `pkg/process` — pulled only by a vo test + non-SDK `pkg/modular`; gone once
  `vo` is rewritten.

Account signing is **write-only pubkey**: the whole SDK touches the account key
at one line (`account.go:18`, `signer.GetPublicKey()`); `Sign()` is never
called. So no account-signature scheme has to be reproduced — only the pubkey
`Any` bytes.

## What must be copied (dependency closure)

From `go list -deps ./uplink-sdk/...`: **~57 packages, ~206 source files, zero
cgo**, minus the drops above. Groups:
- **uplink-sdk roots:** the 6 top-level `*.go` + `internal/` (splitter,
  segmentupload, segmentdownload, pieceupload, piecedownload, streambatcher,
  workerclient, metrics, dial, benchutil) + `cmd/*` + `test/integration_test.go`.
  (`internal/scheduler` dropped — see simplification.)
- **depin `internal/*`:** vo (rewritten), grpcerr, grpcutil/* (connector,
  connector/p2p, cache, grpcconn, peer, pool), p2putil, p2pmonitor, peertls (+
  extensions, tlsopts), fpath, netutil, memory, sync2.
- **depin `pkg/*`:** pb/coord/{client,file,placement,storage}/v1, pb/shared,
  pb/worker/piece/v1, common (+ errs2, pkcrypto, sync2, sync2/race2, time2),
  identity (minus key_manager.go), eestream, infectious, encryption, ers_schema,
  ranger, readcloser, signing, pkcrypto, merkle. (`pkg/process` dropped.)

## Approach

Keep depin's directory layout under `go-sdk/` so the import rewrite is a pure
prefix substitution and re-syncing stays cheap. Drive the copy with a
checked-in script; then apply the three targeted rewrites (de-couple, simplify,
log) as isolated, reviewable diffs.

### Module & layout

```
go-sdk/                     module aioz-depin/go-sdk   (go 1.25)
  client.go options.go upload.go download.go account.go placement.go   ← package uplinksdk (from uplink-sdk/*.go)
  internal/   vo (rewritten), grpcerr, grpcutil/*, p2putil, p2pmonitor,
              peertls/*, fpath, netutil, memory, sync2, + splitter,
              segmentupload, segmentdownload, pieceupload, piecedownload,
              streambatcher, workerclient, metrics, dial, benchutil
  pkg/        pb/*, common/*, identity, eestream, infectious, encryption,
              ers_schema, ranger, readcloser, signing, pkcrypto, merkle
  cmd/        upload-demo, download-demo, sdk-benchmark, pipeline-benchmark
  test/       integration_test.go
  hack/sync-from-depin.sh
  go.mod  go.sum  Makefile  README.md
```

Copied `internal/*` stays internal to `go-sdk`; external consumers import only
the root `aioz-depin/go-sdk` package.

### Import-rewrite rule (single prefix map, ordered)

1. `aioz-depin/uplink-sdk/internal/X` → `aioz-depin/go-sdk/internal/X`
2. `aioz-depin/uplink-sdk`            → `aioz-depin/go-sdk`
3. `aioz-depin/internal/X`            → `aioz-depin/go-sdk/internal/X`
4. `aioz-depin/pkg/X`                 → `aioz-depin/go-sdk/pkg/X`

(uplink-sdk `internal/*` names don't collide with depin-root `internal/*`.)
Non-`aioz-depin` imports untouched.

### De-couple cosmos (the publishability work)

1. **Rewrite `internal/vo` (3 files) with public crypto:**
   - `privkey.go` — parse `priv_key.json` (`{"@type":".../ethsecp256k1.PrivKey",
     "key":"<base64 32-byte scalar>"}`); expose `Bytes()` (32-byte scalar, kept
     identical for identity HKDF) and `GetPublicKey()`. Derive the compressed
     33-byte pubkey via `github.com/ethereum/go-ethereum/crypto` (or
     `btcec`/`decred secp256k1`).
   - `publickey.go` — reimplement the gogoproto `customtype` interface
     (`Marshal/Unmarshal/MarshalTo/Size/MarshalJSON/UnmarshalJSON/Equal`) to emit
     the exact ethermint `Any`: constant `type_url` + value `0a 21 ‖ 33 bytes`.
     **No cosmos codec/registry** — hardcode the string, hand-encode the bytes.
   - `address.go` — `AccAddress` as a 20-byte blob; `String()` = EVM hex via
     `go-ethereum/common`. Drop `sdk.GetConfig`/bech32 helpers (or use public
     `github.com/cosmos/btcutil/bech32` if bech32 parsing is ever needed).
   - Delete `codec.go` and the global `Marshaller`.
   - Fix `address_test.go` (drop `pkg/process` import).
2. **Drop** `pkg/identity/key_manager.go` and `pkg/process`.
3. **Keep** the generated `pkg/pb/*` verbatim — they compile against the
   rewritten `vo` because it satisfies the same customtype method set.

### Simplify: remove multi-worker fan-out

Two concurrency layers exist; only the second is removed:
- **Segments** — already sequential (`upload.go` uses `scheduler.New(1)`).
- **Pieces within a segment** — concurrent fan-out via `pieceupload.Manager` +
  `Client.pieceConcurrency`. This is the "multiple worker" to drop.

Changes: delete `internal/scheduler`; drop the `pieceConcurrency` field,
`WithPieceConcurrency`, `SetPieceConcurrency`; collapse `internal/pieceupload`
and `internal/piecedownload` to a **sequential loop** (dial worker → put/get
piece → next), keeping RS encode/decode + merkle/commit unchanged; trim
`WithResourceLimits`/resource-manager tuning that only mattered for heavy
fan-out (the `MaxConnsPerIP=512` workaround in `client.go` becomes moot with one
in-flight dial). Trade-off: slower per segment, accepted for simplicity +
traceability.

### Debug logging

Thread `zap` `Debug` lines (logger already on `Client.logger`, default
`NewNop`, opt-in via `WithLogger`), keyed so one transfer is followable end to
end: coord RPCs (CreateFile/BeginSegment/CommitSegment/CommitFile/
GetDownloadInfo), per segment (number, size, RS-vs-inline), per piece (index,
worker id/addr, dial start/done, bytes put/got, RS enter/exit, merkle root).
Model field conventions on `storj.io/uplink`.

### Steps

0. **Read `../uplink`** (`upload.go`, `download.go`, `project.go`,
   `private/piecestore`) for API shape + tracing conventions.
1. **`hack/sync-from-depin.sh`** — rsync the closure package list into `go-sdk/`
   (moving `uplink-sdk/*.go` to root, `uplink-sdk/internal/*` under `internal/`),
   run the ordered prefix rewrite + `gofmt`. Excludes `key_manager.go`,
   `pkg/process`, `internal/scheduler`. Re-runnable.
2. **`go.mod`** — `module aioz-depin/go-sdk`, `go 1.25`. Start from depin's
   `require` block **minus** all cosmos/ethermint/ibc-go/private lines and
   **with no `replace`s**; add `go-ethereum` (+ secp256k1 lib) as direct deps;
   then `GOFLAGS=-tags=purego go mod tidy`. Verify `go mod graph` contains no
   `cosmos`/`ethermint`/`10.0.0.50` node.
3. **De-couple cosmos** (rewrite 3 vo files, drop the 2 files/pkg) until
   `go build -tags purego ./...` is green.
4. **Simplify** (remove fan-out) and **add debug logs** as isolated diffs.
5. **`Makefile`** (`build`/`test`/`tidy` pass `-tags purego`) + **README**
   (real usage, `purego` note, sync-script note; drop stale stub language).

## Critical files

- Source: `depin/uplink-sdk/*.go`, `uplink-sdk/internal/*`, and closure packages
  under `depin/internal/*`, `depin/pkg/*`.
- Rewrite targets: `internal/vo/{privkey,publickey,address}.go` (+ delete
  `codec.go`); drop `pkg/identity/key_manager.go`, `pkg/process`,
  `internal/scheduler`.
- Reference: `../uplink` (`storj.io/uplink`).
- New: `go-sdk/hack/sync-from-depin.sh`, `go.mod`, `Makefile`, `README.md`.

## Verification (end-to-end)

1. **No private deps:** `go mod graph | grep -E 'cosmos|ethermint|ibc-go|
   10\.0\.0\.50'` returns nothing. This is the publishability gate.
2. **Builds/vet:** `go build -tags purego ./...` and `go vet -tags purego ./...`
   from `go-sdk/`.
3. **Pubkey wire-parity (the risky item):** unit-test the rewritten
   `vo.PublicKey` — load a real `priv_key.json`, marshal `RegisterRequest`, and
   **byte-diff** the `public_key` `Any` against bytes produced by the original
   depin code (golden vector). Must be identical.
4. **Tests:** `go test -tags purego ./...` (copied `download_internal_test.go`,
   package tests) pass unchanged.
5. **Integration (real flow):** run `test/integration_test.go` and the demo
   binaries against a coord with `WithLogger` at debug:
   ```
   go run -tags purego ./cmd/upload-demo   -identity-dir <dir> -coord-url <id@host:port> -file hello.txt
   go run -tags purego ./cmd/download-demo -identity-dir <dir> -coord-url <id@host:port> -file-id <uuid>
   ```
   Round-trip a multi-segment file: RegisterAccount succeeds (proves pubkey wire
   parity live), download bytes equal source, and debug logs show one
   segment / one piece at a time (proves fan-out removed).

## Out of scope / follow-ups

- Inverting the dependency (depin importing go-sdk) — later refactor; this plan
  duplicates via sync script.
- Final public module path / domain (currently `aioz-depin/go-sdk` on internal
  GitLab) — rename before any real public release.
