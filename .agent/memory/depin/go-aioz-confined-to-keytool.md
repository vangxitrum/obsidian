# go-aioz confined to cmd/keytool

Branch `refactor/drop-go-aioz-dependency` (uncommitted as of 2026-08-12).

## What it was

`pkg/encoding/aioz_encoding.go` booted a **full AIOZ chain application**
(`aioz.NewAIOZApp` + in-memory DB, every module: bank/staking/gov/ibc-go/ethermint
app/Gravity-Bridge/cometbft/simulation) just to read `.AppCodec()`. It was reached from a
package-level `init()` in `worker/pkg/keytool/cosmos_keytool.go`, so **every** binary that
touches keys linked the whole chain.

The codec was used by exactly ONE method: `GetPrivKeyFromJSON`.

## The fix

`internal/vo/codec.go MakeCodec()` (std + ethermint interfaces) already existed **and the
verifying side already used it** - `versioncontrol/api/handler.go:47` via
`vo.NewPublicKeyFromString`. So client and server were already interoperating across the
two codecs in production: proof the wire format could not change.

`CosmosKeyTool` got an optional `Codec` field (nil -> `sync.OnceValue(vo.MakeCodec)`).
`cmd/keytool/keyprovider.go` passes the AIOZ codec via `sync.OnceValues`, lazily. Zero
churn at worker/updater call sites - `&keytool.CosmosKeyTool{}` still compiles.

## Measured (stripped -s -w)

| binary | before | after |
|---|---|---|
| worker | 181.5M | **56.9M** (-69%) |
| worker-updater | 174.2M | **47.6M** (-73%) |
| keytool | 159.5M | 159.6M (keeps go-aioz by design) |

`go list -deps ./cmd/worker ./cmd/worker-updater | grep AIOZNetwork/go-aioz` == 0.
Package graph 2286 -> 1353. Updater's auto-update zip 58.8M -> ~17M.

## Traps found

- **`linux/arm` was never really blocked by keytool alone.** Removing the `blocked` marker
  is NOT enough: the `binaries` **group** lists targets explicitly and had no
  `binaries-linux-arm`. Must add it or arm silently stays unbuilt.
- Images are unaffected by the arm change: `_image_common` pins
  `platforms = ["linux/amd64","linux/arm64"]`, so `cmd/worker/Dockerfile`'s unconditional
  `COPY .../keytool` never runs 32-bit.
- **`cmd/worker/Dockerfile` ships worker+updater+keytool.** Keeping go-aioz in keytool
  leaves ~160M of it in the public worker image; image binary payload 514M -> 264M instead
  of 144M. A separate `keytool-image` target exists if that ever matters.
- keytool cannot cross-compile with `CGO_ENABLED=0` (ethermint `secp256k1.RecoverPubkey`).
  Pre-existing; bake uses `cgo = "1"` everywhere via zig. Don't "fix" it.

See [[keytool-secrets-and-aioz-prefix]] for the CLI changes shipped alongside.
