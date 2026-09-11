# worker-console

Local HTTP API for the DePIN Worker GUI, on branch `feat/worker-local-httpserver-for-UI`
(uncommitted, 2026-08-28).

## What and why

The depin worker had **no HTTP server at all** — only the generic debug server bound to
`127.0.0.1:0` (an ephemeral port, so unaddressable). The legacy AIOZ IO worker
(`go-aioz-media`, `worker/rest/root.go`) does have one: a local REST router on
`tcp://localhost:1317` that an operator GUI talks to. This ports the operator half of
that router onto depin's subsystems.

New package `worker/console` + 58 lines in `worker/peer.go`. Seven routes under
`/api/v0`, all off unless `--worker.console.enabled`:

| route | ported from |
|---|---|
| `GET /ping` | `/ping` |
| `GET /node` | `/ui/node_info` |
| `GET /storage` | `/config/get_storage_config` + storage half of `/stats` |
| `POST /storage/limit` | `/config/set_storage_config` |
| `GET /payout/balance` | `/worker/balance` |
| `GET /payout/withdrawals` | new (legacy could file one, never show one) |
| `POST /payout/withdrawals` | `/worker/withdraw` |

Not ported: `/ui/task_status` (no task manager), `/worker/register` (identity+trust
replaces it), `/testnet_migration/*`, `/config/set_delivery_config` (a stub),
`/debug/*` + `/metrics` (already on the debug server), all public data-plane routes.

## Security decisions

No authentication — the listen address is the boundary, default `127.0.0.1:7780`, same
model as coord/admin. Deliberately sends **no CORS headers**: go-aioz-media set
`AllowedOrigins: *` on its unauthenticated local router, meaning any page the operator
visited could call every route, which is why a signed-token check was later bolted onto
its `/worker/withdraw` alone. No CORS header closes that for all routes at once. A GUI
on another origin uses a dev-server proxy (coord/admin/ui/vite.config.ts pattern).

## Traps worth remembering

- **Generated protobuf structs are all `omitempty`.** Embedding `workerpb.DiskSpace` in a
  JSON response made genuine zeros vanish from the payload — a GUI cannot tell "zero"
  from "not reported". Restate fields explicitly. Unit tests with non-zero fakes do not
  catch this; the testplanet e2e did.
- **`worker/pkg/space/shared.go` declares `var storageStatus StorageStatus` and never
  populates it**, so `Total` and `Free` are always 0 on real nodes, not just in tests.
  Any capacity check written against `Total` is inert. Pre-existing; not fixed here.
- **`cmd/worker/config_tool.go` has never worked**: writes `node.storage-limit` (maps to
  `config.AppConfig.StorageLimit`, which nothing reads) in *nested* YAML, while
  `process.SaveConfig` emits *flat* dotted keys; and its `WorkerConfigCmd()` is never
  registered on rootCmd. Real key is `worker.storage.allocated-disk-space`.
- **`process.SaveConfig` needs a live `*cobra.Command`**, so a running daemon cannot
  reuse it to edit config — hence a hand-rolled flat-key line editor.
- **Bech32 needs `process.RegisterBlockchainDefaultCfg()` in an init.** A test binary
  without it rejects every valid bech32 address as malformed.
- **Dial coordinators in-daemon** via `trust.GetNodeURL` + `peer.Dialer.DialNode`, never
  `worker/payoutclient` — that exists for one-shot CLIs avoiding the daemon's DB and
  bbolt locks.
- `POST /storage/limit` cannot take effect live: allocation is read once at startup, so
  the response carries `restart_required: true`.
- **Setting allocated BELOW used is allowed, not an error.** Upstream Storj
  (`storagenode/monitor/shared.go:131`) logs "Used more space than allocated" and
  continues, because shrinking below current usage is the only way to drain a node.
  Consequence: `available` clamps to 0, check-in advertises no free space, and
  `worker/piecestore/endpoint.go:271` refuses uploads ("not enough available disk
  space"). Existing pieces are KEPT and still served, so the node keeps earning while
  GC/expiry bring usage down. The console returns `used`/`overused`/`warning` because
  depin's `SharedDisk.PreFlightCheck` is `return nil` -- Storj's whole 58-line preflight
  (the overused warning, free-disk capping, below-minimum startup refusal) was dropped
  in the port, so there is no startup log to fall back on.

## Verification

`go build ./...`; 30 unit tests green under `-race`; golangci-lint 0 issues; all
`worker/...` packages pass; full `./internal/testplanet/` suite green (381s, 0 failures);
`TestWorkerConsole` e2e (real peer + real coord over real HTTP, 8 subtests) and
`TestWorkerConsoleDisabledByDefault` both pass.

See [[depin-test-postgres]] for how to actually run the suite.
