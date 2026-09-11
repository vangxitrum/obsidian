# keytool: aioz1 prefix + secrets off argv (and 3 real bugs found)

Branch `refactor/drop-go-aioz-dependency`, alongside [[go-aioz-confined-to-keytool]].
Uncommitted as of 2026-08-12.

## aioz1, not cosmos1

`cmd/keytool/main.go` never called `process.RegisterBlockchainDefaultCfg()`, which
`cmd/worker/main.go:81` and `cmd/coord/main.go:128` both do. So **keytool and coord
disagreed**: keytool printed `cosmos1...`, `coord/billing/endpoint.go:82` returned
`aioz1...` for the same key.

One line in `main()` (NOT `init()` - it ends in `sdkConfig.Seal()`, one-shot).

**Derived keys do not move**, proven with a fixed mnemonic before/after: `address_hex`,
`pub_key`, `priv_key` byte-identical, only the bech32 prefix changed (same data part
`jrkmdcwgq94uaamx6zax2luewlhf7u4k`). The call also sets the global coin type to
ethermint's 60, but `worker/pkg/keytool/keytool.go:28 CoinType = 118` is passed
**explicitly** to `hd.CreateHDPath`, so the global never reaches derivation. depin
deriving on 118 while the chain declares 60 is a pre-existing inconsistency - do not
"fix" it, it would change every address.

Nothing persisted changes: `vo.AccAddress.Value()` stores the **hex** form.

## Secrets off the command line

Six surfaces, and the worst were **positional args**, not flags: mnemonic (`recover`),
raw private key JSON (`encrypt`), armor (`decrypt`/`sign`), plus `create --mnemonic`.

Two channels, so payload and password never compete for stdin:
- payloads: `--mnemonic-file` / `--key-file` / `--armor-file`, else all of stdin
- password: `--password-file`, else no-echo prompt on **`/dev/tty`**, else one line of stdin

`/dev/tty` matters: cosmos's `input.GetPassword` uses speakeasy, which reads **os.Stdin**
(stty echo off), so it would eat the payload. Used `golang.org/x/term` on `/dev/tty`
instead (already in the graph; promoted to a direct dep in go.mod).

Output: stdout carries only `address`/`address_hex`/`pub_key`. Key material goes to
`--secrets-out <path>` (0600), else stderr when it is a TTY, else withheld with a message.
`keytool.Response` was **split** into `Response` (public) + `SecretResponse` so a stray
`zap.Any` has no field to leak.

Fleet automation unaffected - `gen-fleet.sh`, `worker-pkg/init.sh`, `rollout-scale-test.sh`
all use `keytool create <name> --difficulty --identity-dir`, none of the changed inputs.

## Four real bugs fixed on the way

The existing `keytool_test.go` drives a **mocked** KeyProvider, so it never reaches real
cosmos crypto - which is exactly how the armor path stayed dead. Added
`cosmos_keytool_test.go` driving `CosmosKeyTool` directly (real round trip, both input
forms, wrong-password, raw-vs-JSON contract, minimal-codec decode).

1. **The whole armor feature was dead.** `runCreateKey`/`recoverKey`/`encryptKey` passed
   `privKeyJSONB` to `EncryptArmorPrivKey`, which protobuf-decodes anything not exactly 32
   bytes -> every `-P` run failed `unexpected EOF`. Must pass `privKeyRawB`.
2. `GetPrivKeyFromAmorKey` did `json.Unmarshal("\"" + armor + "\"")`, i.e. it only accepted
   the **JSON-escaped** armor copy-pasted out of the printed response. Broke on real
   multi-line armor from a file. Now `normalizeArmor` accepts both forms, discriminating
   on **line structure** (a real block contains newlines; the escaped form is one line).
   Do NOT discriminate on the `-----BEGIN` prefix - BOTH forms start with it - and do NOT
   use "did strconv.Unquote succeed", which mangles any armor containing a backslash.
3. **Armor needs the global legacy Amino codec to know eth_secp256k1**, and that used to
   arrive as a side effect of building the AIOZ chain app. With the app gone from the
   default path, `legacy.Cdc.MustMarshal` produced prefix-less bytes and unarmor died with
   `unmarshal to types.PrivKey failed after 4 bytes (unrecognized prefix bytes ...)`.
   Fixed with a lazy `sync.OnceFunc` calling `ethermintcodec.RegisterCrypto(codec.NewLegacyAmino())`
   (which itself reassigns the `legacy.Cdc` global) before both armor directions. This is
   the codec-divergence risk actually materialising - worth remembering that anything
   Amino-encoded has the same hidden dependency.
4. `decrypt` printed the **hex** address in the `address` field while every other command
   printed bech32.

## pkg/auth replay window (separate blast radius)

`pkg/auth/auth_http.go:50` was `TimestampWithin(tsInt, 3600000000000)` - nanoseconds into a
parameter measured in **seconds** then multiplied by `time.Second` again = ~114,000 years,
so the replay check was **disabled**. Also overflowed 32-bit int and broke `linux/arm`.
Now `3600` via a named const.

`EthermintVerifier.Verify` is used by **both** `versioncontrol/api/middlewares.go` and
`coord/server/http_middlewares.go`, so this tightens auth **fleet-wide**: >1h clock skew
now fails. Ship as its own revertable commit.

## Verified

Full `internal/testplanet` suite green (523s) against Postgres - this is what clears the
gogo-proto registry-asymmetry risk (the "duplicate proto type registered" lines are
pre-existing log noise, not a panic). `linux/arm` now cross-compiles for worker+updater.

**Trap:** testplanet `Run()` silently `t.Skip`s without `DEPIN_TEST_POSTGRES` and the suite
exits 0 - it looks green while running nothing. Also, `go test ./...` against ONE scratch
DB fails `TestRecordPeriodTwiceChargesOnce` on concurrent `CREATE EXTENSION uuid-ossp`;
passes in isolation, environmental not a regression.
