---
type: fact
tags: [depin, coord, order, monkit, metrics, testing]
created: 2026-07-09
agent: task-5-subagent
---

Task 5 (commit `8548ebe`, branch `feat/coord-metrics-collection`, same branch
as [[coord-audit-metrics-instrumentation]] and
[[coord-repair-metrics-instrumentation]]): added monkit counters for order
signing and settlement completion in `coord/order`.

**Metrics added:**
- `order_limits_signed` / `order_limits_signing_errors` — Counters tagged
  `action=<PieceAction.String()>` (PUT, GET, GET_AUDIT, GET_REPAIR,
  PUT_REPAIR). Placed in `coord/order/signer.go`'s `Signer.Sign()` method —
  this is the *one* real per-piece signing outcome site shared by every
  `Service.Create*OrderLimits` caller in `service.go`, not the wrapper
  methods themselves (those are closer to request entry).
- `order_settlement_bytes` (IntVal) / `order_settlement_rows` (Counter) —
  emitted in `coord/order/settlement.go`'s
  `SettlementEndpoint.SettlementWithWindow`, per successfully-completed
  rollup row (right after `store.AddWorkerBandwidthSettle` succeeds for that
  (worker, action, hour) key), not at stream/request entry. **Fixed
  (commit `448ec8d`, same branch)**: now tagged `action=<PieceAction.String()>`
  too — reviewer caught that they were originally left untagged even though
  `verifyOrder` admits both GET and GET_AUDIT through this one site, and the
  per-row `rollupKey.action` was already available to tag with (no new
  aggregation needed, same tag-value convention as the signing side).

**Key finding:** `coord/order/endpoint.go` is nearly empty in this codebase
(just `Error`/`ErrUsingSerialNumber` classes + the package `mon` var) — it is
NOT where endpoint/RPC logic lives, despite the name. `OrdersService`'s only
RPC (`SettlementWithWindow`) is implemented in `settlement.go`. Don't assume
`endpoint.go` holds real logic in this repo without checking.

**Tag pattern confirmed:** `worker/piecestore/endpoint.go:651`
(`monkit.NewSeriesTag("action", limit.Action.String())` on shared counter
names like `download_started_count`) is this codebase's idiomatic
"one counter, many kinds via tag" pattern — now also used in `coord/order`
for signing. Prefer this over minting N separately-named counters when the
"kinds" are variants of the same event.

**Test patterns:**
- Reused [[coord-audit-metrics-instrumentation]]'s `withTestMon`
  (swap package `mon` var for a `monkit.NewRegistry()` scope, restore on
  cleanup) but generalized `assertCounter` to take `[]monkit.SeriesTag` since
  `order`'s counters are tag-based (audit/repair's counters were untagged).
- To drive `SettlementWithWindow` end-to-end in a test (real gRPC-streaming
  handler, not a mock), needed: a hand-rolled fake implementing
  `grpc.ClientStreamingServer[Req,Resp]` (Recv/SendAndClose/SetHeader/
  SendHeader/SetTrailer/Context/SendMsg/RecvMsg), plus
  `grpcpeer.NewContext(ctx, &grpcpeer.Peer{State: tls.ConnectionState{
  PeerCertificates: ident.Chain()}})` from `internal/grpcutil/peer` to inject
  a real mTLS peer identity so `identity.PeerIdentityFromContext` resolves
  for real (that package's `FromContext` checks a context-value seam first,
  before falling back to real grpc `peer.FromContext`+TLSInfo — deliberate
  test seam). This pattern is reusable for testing any `coord` endpoint that
  resolves worker identity from `ctx`.

All `coord/order` tests green (build/vet/gofmt/`test -race` all clean). 3
pre-existing unrelated build failures elsewhere in the repo
(`uplink-sdk/cmd/download-demo`, `internal/testplanet`) confirmed via
`git stash` to predate this task.

**Review-fix follow-up (commit `448ec8d`, same branch):** reviewer found 2
Important issues, both fixed:
1. The `continue` in the per-rollup-row loop that skips recording bytes/rows
   counters when `AddWorkerBandwidthSettle` fails was untested — both existing
   tests used a `fakeSettlementStore{}` with a zero `err`, so the failure
   branch never ran. Fix: added `fakeSettlementStore.errFor map[int32]error`
   for per-action selective failure, and a test
   (`TestSettlementWithWindow_StoreWriteFailure_SkipsCountersForFailedRow`)
   with one GET row that succeeds and one GET_AUDIT row whose store write
   fails in the same window — proves the `continue` is scoped per-row, not
   per-batch (the successful sibling row's counters still increment).
2. Settlement counters were untagged (see above) — fixed with an
   `action`-tagged `monkit.SeriesTag`, plus a
   `TestSettlementWithWindow_ActionTagging_GetVsGetAudit` test mirroring
   `signer_metrics_test.go`'s tag-isolation test pattern.
`intValSum` test helper (`coord/order/monkit_test.go`) now takes variadic
`...monkit.SeriesTag` so tagged IntVal series can be read independently,
mirroring `assertCounter`'s existing tags param.
