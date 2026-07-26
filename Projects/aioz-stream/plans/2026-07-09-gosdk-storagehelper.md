# Plan: go-sdk-backed StorageHelper implementation

## Context
aioz-stream's storage layer is defined by the `StorageHelper` interface
(`internal/utils/storage/storage.go:11`) and currently has one implementation,
`CdnHelper` (`internal/utils/storage/cdn.go`), which talks to the AIOZ CDN + Hub
over HTTP. We want a second implementation backed by the local go-sdk
(`~/work/depin-workspace/go-sdk`, module `aioz-depin/go-sdk`, package `uplinksdk`) —
AIOZ DePIN decentralized storage (mTLS + gRPC, client-side AES-GCM + erasure coding).

**Key impedance mismatch (decided with user):** go-sdk only backs upload, download,
and a download-ticket (presigned analog). It has **no** delete, transcode, zip, price,
balance, or file-record concepts. Per user decision:
- Unsupported methods **return `ErrNotImplemented`**.
- `Object` maps as **`Id = uuid.String()`, `Offset = 0`**, `Size` from upload size.
- Scope = **impl + wiring + config** (add backend switch at startup).

## Files to change

### 1. New impl: `internal/utils/storage/gosdk.go`
New type `GoSdkHelper` satisfying `StorageHelper`, wrapping `*uplinksdk.Client`.

Constructor:
```go
func MustNewGoSdkHelper(identityDir, coordPeerURL, pieceKeyPath, linkEndpoint string, opts ...GoSdkOption) StorageHelper
```
Internally builds the client (`uplinksdk.New(ctx, WithIdentityDir, WithCoordPeerURL,
WithPieceKey(pieceKeyPath), WithLogger)`), stores `linkEndpoint` on the struct for
`GetLink`, panics on error (mirrors `MustNewCdnHelper`, `cdn.go:61`).

Method mapping (go-sdk API from `upload.go` / `download.go`):

| StorageHelper method | go-sdk backing |
|---|---|
| `Upload(ctx, data, reader)` | buffer reader → `UploadFile(ctx, buf, UploadParams{Filename:data, Size:n})`; return `&Object{Id: uuid.String(), Offset:0, Size:n, Name:data}` |
| `PackUploadByte(ctx, name, data)` | `UploadFile` with `bytes.NewReader(data)`, `Size=len(data)`, `Name=name` |
| `UploadZip(ctx, data, reader)` | same as `Upload` (no zip packing; whole file) |
| `UploadRaw(ctx, data, size, reader)` | `UploadFile(ctx, reader, UploadParams{Filename:data, Size:size})` — size already known, no buffering |
| `Uploads(ctx, data, fileInfos)` | loop each `io.Reader`, `UploadFile` each; return `[]*Object` + total size |
| `Download(ctx, object)` | `io.Pipe`: goroutine runs `DownloadFile(ctx, uuidFromString(object.Id), pw)`; return `pr` |
| `GetLink(ctx, object)` | `CreateDownloadTicket(ctx, uuid, expiresAt)`; build a playable URL from a **configurable link endpoint** + the base64 ticket: `{linkEndpoint}/file/{id}?ticket={base64}&expire={unix}`. Return that URL string + expiry unix. Endpoint comes from config (`GOSDK_LINK_ENDPOINT`) / a `WithLinkEndpoint` option. |
| `GetDetailBalance(ctx)` | `GetAccount(ctx)` → return `&GetDetailBalanceResponse{DepositAddress: account.Address.String()}` (credit/expense fields left empty — go-sdk has no credit ledger). Fills the business/deposit address so callers that validate it still work. |
| `Delete`, `Transcode`, `GetTranscodeStatus`, `GetZipHeader`, `GetFileRecord`, `GetAIOZPrice` | `return ..., ErrNotImplemented` |

Add at top of file:
```go
var ErrNotImplemented = errors.New("storage: operation not supported by go-sdk backend")
```

**UUID round-trip gotcha:** `UploadFile` returns `vo.UUID` (`[]byte`) but `vo` is an
`internal/` package — not importable. Store `uuid.String()` in `Object.Id`; to
download, reconstruct via the SDK's public helper. Verify go-sdk exposes a public
`UUID`-from-string (e.g. `uplinksdk.ParseUUID` / hex-decode). **If none exists,
this is a blocker** — resolve during implementation (may need a tiny exported
helper added to go-sdk, since `DownloadFile`/`CreateDownloadTicket` take `vo.UUID`).

**Size gotcha:** `UploadParams.Size` is required up front, but `Upload`/`UploadZip`
get only an `io.Reader`. Buffer fully into memory to measure (acceptable for now;
note memory cost for large files). `UploadRaw` avoids this (size passed in).

### 2. Config: `internal/config/config.go` (after line 54)
```go
StorageBackend    string `mapstructure:"STORAGE_BACKEND"`      // "cdn" (default) | "gosdk"
GoSdkIdentityDir  string `mapstructure:"GOSDK_IDENTITY_DIR"`
GoSdkCoordPeerURL string `mapstructure:"GOSDK_COORD_PEER_URL"`
GoSdkPieceKeyPath string `mapstructure:"GOSDK_PIECE_KEY_PATH"`
GoSdkLinkEndpoint string `mapstructure:"GOSDK_LINK_ENDPOINT"`  // base URL GetLink builds playable links from
```

### 3. Wiring: `cmd/http/init.go:484` and `cmd/grpc/init.go:81` (identical blocks)
Replace the `storageHelper = storage.MustNewCdnHelper(...)` call with a switch:
```go
switch appConfig.StorageBackend {
case "gosdk":
    storageHelper = storage.MustNewGoSdkHelper(
        appConfig.GoSdkIdentityDir, appConfig.GoSdkCoordPeerURL,
        appConfig.GoSdkPieceKeyPath, appConfig.GoSdkLinkEndpoint)
default:
    storageHelper = storage.MustNewCdnHelper(
        appConfig.CdnUrl, appConfig.HubUrl, appConfig.BusinessAddress)
}
```
Default (empty / "cdn") preserves current behavior — no env change breaks prod.

### 4. Module deps: `go.mod`
- `require aioz-depin/go-sdk v0.0.0` + `replace aioz-depin/go-sdk => /home/tuan/work/depin-workspace/go-sdk` (local, unpublished GitLab repo).
- Run `go mod tidy` to pull go-sdk's transitive deps (libp2p, zap, protobuf, etc.) — sizable dep tree.
- **Build-tag caveat:** go-sdk requires `-tags purego` (missing AMD64 asm in `pkg/infectious`). Adding it as a dep means aioz-stream's build/test/CI commands must add `-tags purego`. Check `Makefile` / CI and update build invocations.

## Verification
1. `go build -tags purego ./...` — compiles with new dep + impl.
2. `go vet -tags purego ./internal/utils/storage/...`.
3. Unit test `internal/utils/storage/gosdk_test.go`:
   - Unsupported methods return `ErrNotImplemented` (table test, no network).
   - `Object` mapping (Id/Offset/Size) — assert with a fake/mock or the go-sdk
     integration harness (`go-sdk/test/integration_test.go`) if a local coord is reachable.
4. End-to-end (needs a running aioz-depin coord + identity dir + piece key): set
   `STORAGE_BACKEND=gosdk` + the three `GOSDK_*` envs, boot `cmd/http`, upload a
   video, confirm `UploadFile` succeeds and `Download` round-trips bytes. If no
   coord is available, gate this behind the integration harness and document env setup.
5. Confirm `STORAGE_BACKEND` unset → still uses `CdnHelper` (regression guard).

## Open items to resolve during implementation
- Public UUID parse helper in go-sdk (blocker for Download/GetLink round-trip).
- `GetLink` semantics: opaque ticket vs playable URL — callers of `GetLink` may
  break under gosdk backend; audit before enabling in prod.
