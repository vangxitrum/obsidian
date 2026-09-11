---
type: decision
tags: [memory, binance, trading, go, security]
created: 2026-09-07
agent: main
---

# Live Binance orders live in a Go sidecar, not the Next.js app

`order/` is a self-contained Go module (stdlib only, no SDK) built to `bin/order` with
`npm run order:build`. It is the only thing in the repo that can spend money. `lib/order.ts`
only spawns it and parses the one JSON object it prints on stdout.

**Why a separate process:** sizing, exchange precision and every guardrail live inside the
binary, so a bug or a tampered web layer cannot place an order the binary would have refused.
The notional cap in particular is checked in Go, not in the route.

## The four guardrails
- `BINANCE_TESTNET` - true unless explicitly `"false"`, including when unset.
- `-dry-run` - signs and prints every request, sends none. Works with no credentials at all.
- the prompt - `-yes` skips it; with no answer it refuses rather than assumes.
- `ORDER_MAX_NOTIONAL_USDT` - checked before any state-changing call. Nothing places without it.

## Non-obvious things learned building it
- **Step size differs between networks.** BTCUSDT futures `LOT_SIZE.stepSize` is `0.001` on
  mainnet but `0.0005` on testnet, and `MIN_NOTIONAL` is 50 USDT. Always read `exchangeInfo`
  from the network you are on; never hardcode precision.
- `FloorToStep` needs an epsilon: `0.003/0.001` is `2.9999999999999996` in binary float, and
  flooring that raw silently drops a whole lot step.
- Binance `-4046` ("No need to change margin type") is the *requested state* reported as an
  error. It must be treated as success or every second order aborts.
- `closePosition=true` + `workingType=MARK_PRICE` on the stop/target: stays correct after a
  partial exit taken by hand, and cannot be knocked out by a last-price wick.
- `newOrderRespType=RESULT` is what makes Binance return the average fill price. Without it the
  journal records the requested price, not what was paid.
- A failure must put its reason in the **stdout JSON**, not only stderr - the caller reads
  stdout, and "exit 1" tells nobody which setting to fix.
- Exit 3 means the entry filled but a protective order did not. The caller must still record
  the position: an open leveraged position nothing knows about is the worst outcome available.

## Security posture (matters, this app has no login)
The app is tunnel-exposed at `order.tunnel.appdemo.cyou` behind an auth gateway and has no
login of its own, so the gateway is the only thing between the internet and a leveraged
position. Defence in depth added: `ORDER_LIVE_ENABLED` defaults false, `/api/orders` requires
`ORDER_API_TOKEN`, and the browser is given only a 30-minute token minted from it - the secret
never reaches the page. Ordering is two-step: preview signs the parameters into a 60-second
token, place verifies it, so the confirmed order is provably the previewed one.

The web path requires a stop and a target (the CLI does not) because `Position.sl`/`tp` are
required columns and an unjournaled position is invisible to the settle job.

Related: [[trade-journal]], [[nextjs-e2e-isolated-copy]]
