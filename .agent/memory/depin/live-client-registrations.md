---
type: reference
tags: [depin, go-sdk, client, registration]
created: 2026-08-03
updated: 2026-08-05
agent: main
---

The Go SDK identity at
`/home/tuan/work/stream/aioz-stream/secrets/gosdk-identity/uplink` was registered
with the live coordinator on 2026-08-03 using `admin@aioz.io`.

The repository identity at
`/home/tuan/work/depin-workspace/depin/identities/uplink` was registered and then
verified with `GetAccount` against
`8ce76c37-fa80-956c-d179-c0db2a000001@tcp:68.183.189.51:7777` on 2026-08-04.
It resolves to the same public client ID and address below; the coordinator returned
`admin@aioz.io` as its email.

- Client ID: `bb010caa-7996-63de-3822-9d9800548d01`
- Address: `0x4d62AF07330bDab17a8F1aa5Dd91035dF47659fC`

Only public account identifiers are recorded here; no private key material is
stored in agent memory.

## 2026-08-05 re-registration

`GetAccount` against the live coordinator returned `NotFound` for this identity
before re-registering, despite the 2026-08-04 verification above - the account
record was gone (coordinator DB reset or client row removed). `RegisterAccount`
recreated it with the **same** client ID and address, because both are derived
from the signer public key in `priv_key.json`, not from any coordinator-side
sequence. Re-registering is therefore safe and idempotent in effect.

Two traps hit on the way:

1. **`cmd/uplink hubclient register` is dead.** It needs package `aioz-depin/uplink`,
   which exists in neither HEAD nor develop, so the binary cannot be rebuilt; the
   `bin/uplink` on disk is a 2026-07-06 artifact. It also parses the key via
   `cmd/uplink/deprecated__keytool.KeyResponse` (`pub_key`/`priv_key` fields), while
   real `priv_key.json` files written by the current keytool are `{"@type","key"}`.
   It fails with `invalid public key value`. The modern path is the go-sdk:
   `uplinksdk.New(WithIdentityDir, WithCoordPeerURL)` then `RegisterAccount(ctx, email)`,
   exactly as `internal/testplanet/uplink.go:228` does it.
2. **Stale `piece_key.json` blocks `New()` entirely**, before any RPC:
   `sdk: load piece key: piece_key.json: piece private key error: invalid private key size`.
   The old format is `{"key": "<base64, 96 bytes>"}`; the SDK wants
   `{"private_key": "<hex, 64 bytes>"}` (go-sdk `client.go:311`). The field name
   doesn't match, so the old material is never even read. Fix per the SDK's own
   comment: delete `piece_key.json` and `loadOrGeneratePieceKey` writes a fresh valid
   one. The repo copy was moved to `identities/uplink/piece_key.json.old-format.bak`.
