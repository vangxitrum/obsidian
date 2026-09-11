---
type: decision
tags: [go-aioz-media, worker, wallet, withdrawals, rest, ui]
created: 2026-09-08
agent: main
---

`GET /worker/withdraw_history` on the worker's local router - cursor-paged
withdrawal history for the UI. Registered next to `/worker/balance` in
`worker/rest/root.go`; wallet routes live under `/worker/`, not `/ui/`.

**Phase 1 (built 2026-09-08, uncommitted): fake data only.**
- `worker/rest/withdraw_history.go` - handler, cursor codec, `withdrawRecord` DTO.
- `worker/rest/fake_withdrawals.go` - 37-record deterministic seed.
- Without `--ui-fake-events` the route answers **501**, deliberately: an empty
  200 would render as "no withdrawals", a different claim from "the hub cannot
  answer yet".
- This is the one exception to the fake mode's events-only rule
  ([[ui-fake-events-flag]]). `/worker/balance` is explicitly NOT faked - the user
  declined it.
- The seed honours `limit`/`cursor` for real. A seed that ignored them would let
  broken UI paging pass unnoticed, which is the whole point of having one.
- Cursor = base64url of `<created_at unix seconds>:<id>`, keyset over
  `(created_at, id)` DESC. The id is in the cursor because withdrawals made in
  one transaction share a `created_at`, and a timestamp-only cursor would skip
  or repeat them.

**Phase 2 (BUILT 2026-09-08, uncommitted): the hub chain.**
`proto/aioz/wallet/v1/{query,wallet}.proto` (new `GetWithdrawHistory` RPC +
`WithdrawRecord`), regenerated `query.pb.go`/`wallet.pb.go`,
`ITxOutStore.GetWithdrawalsByOwner` keyset query, `hub/wallet/withdraw_history.go`
(querier + controller method + cursor codec), `GrpcClient.GetWithdrawHistory`,
and the worker handler's live branch. Hub cursor uses UnixNano (Postgres keeps
sub-second precision, and two rows in one transaction differ only below the
second); the worker forwards the hub's cursor verbatim rather than re-encoding.
Querier uses a POINTER receiver - the value receiver used by its neighbours
copies a `semaphore.Weighted` and trips vet.

Original background:
No withdrawal-history source exists today. The hub wallet `Query` service has 7
RPCs and none returns withdrawals (`GetDepositHistory` is inbound `TxIn`).
`WorkerWallet.widthdraw` (typo is in the proto) is a cumulative total only.
The data does exist: `Controller.WithdrawWorkerWallet`
(`hub/wallet/controller.go:369`) writes an `InternalTransaction{Type:
TxTypeWithdraw}` plus a `TxOut` row carrying From/To/Amount/Txid/Status/CreatedAt.
Work needed: proto RPC -> `ITxOutStore` keyset query by owner -> hub querier
(copy `GetWorkerWallet`'s `connect.UnwrapAuthContext` + `RoleWorker` pattern, so
the request needs no owner field and a worker cannot read another's history) ->
`grpc_client` wrapper -> the handler's non-fake branch.

**Proto regeneration: SOLVED 2026-09-08, and verified byte-for-byte.** It is not
a blocker. Recipe (nothing committed to the repo):

1. `go install github.com/cosmos/gogoproto/protoc-gen-gocosmos@v1.4.10` - this is
   the plugin that actually produced the committed pb.go files. `protoc` itself is
   NOT needed; `buf` (already on PATH) brings its own compiler.
2. Assemble a temp proto root outside the repo: copy `proto/aioz` plus
   `$(go env GOMODCACHE)/github.com/evmos/cosmos-sdk@v0.47.12-evmos.2/proto/*`,
   `cosmos/gogoproto@v1.4.10/gogoproto/gogo.proto`, and
   `cosmos-proto@v1.0.0-beta.5/proto/cosmos_proto/cosmos.proto`.
3. `buf.gen.yaml` with plugin `gocosmos`, opts
   `plugins=interfacetype+grpc` and
   `Mgoogle/protobuf/any.proto=github.com/cosmos/cosmos-sdk/codec/types`.
4. `buf generate --path aioz/wallet/v1 --output <tmp>`.

Regenerating `query.pb.go`, `wallet.pb.go` and `workerwallet.pb.go` this way
reproduces the committed files **identically** (zero diff), so the toolchain is
confirmed correct.

**Both checked-in generation entry points are stale and would produce WRONG
output.** `scripts/protocgen.sh` targets regen-network protobuf + cosmos-sdk
0.42.6 (go.mod is on 0.47.12-evmos.2) and `proto/Makefile` hardcodes
`/home/trieu/...` and only covers `aioz/connect/v1`. The committed pb.go imports
`github.com/cosmos/gogoproto/...`, which neither script would emit. Repairing
them is worthwhile cleanup but is NOT a prerequisite for adding an RPC.

**Also needed for phase 2:** a composite index on `tx_outs (from, created_at
DESC, id DESC)`. Today's `idx_tx_outs_from` and `idx_tx_outs_created_at` are
separate and cannot serve that ordering. Create it `CONCURRENTLY` out of band,
not via gorm automigrate at hub startup.

See [[ui-fake-events-flag]] and [[worker-ui-event-stream]].
