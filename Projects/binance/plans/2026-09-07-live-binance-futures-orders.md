---
tags: [project, binance, trading, go, nextjs]
created: 2026-09-07
status: shipped (pending testnet order + crontab update)
---

# Live Binance futures orders (x5) via a Go sidecar

Renamed the repo `trace` -> `binance` (`/home/tuan/personal/binance`) and gave it the ability to
actually place the trade it has only ever described.

## Context

The app was a read-only journal: it captured Binance public market data hourly, derived a plan
with three gates, watched for price entering the zone, settled positions from klines, and let
you record by hand a trade you had placed yourself in the Binance UI. That hand-off is where
slippage and missed fills came from. There was no signing, no API key and no order placement
anywhere in the codebase.

## What shipped

**`order/` - a Go sidecar** (stdlib only, no SDK, built to `bin/order` via `npm run order:build`).
The only thing in the repo that can spend money. Sequence: set leverage -> set margin type
(isolated, tolerating Binance `-4046`) -> MARKET entry with `newOrderRespType=RESULT` so the
fill price comes back -> `STOP_MARKET` and `TAKE_PROFIT_MARKET`, both `closePosition=true` off
the mark price -> position readback. Prints one JSON object on stdout.

Why a separate process: sizing, exchange precision and every guardrail live in the binary, so a
bug or a tampered web layer cannot place an order the binary would have refused.

**Four guardrails**, all enforced in Go before any state-changing call:

| | |
|---|---|
| `BINANCE_TESTNET` | true unless explicitly `"false"`, including when unset |
| `-dry-run` | signs and prints every request, sends none; works with no credentials |
| the prompt | `-yes` skips it; with no answer it refuses rather than assumes |
| `ORDER_MAX_NOTIONAL_USDT` | ceiling on one order's notional; nothing places without it set |

**`lib/order.ts`** spawns the binary (array args, no shell, env allowlist, 20s timeout) and
parses its JSON. **`app/api/orders/route.ts`** is two-step: preview runs the binary in dry-run
and signs the exact parameters into a 60-second token; place verifies that token, so the order
that reaches Binance is provably the one whose numbers were on screen. On a fill it records a
`Position` at the **actual fill price** and pushes the chart overlay. **`PositionForm.tsx`**
gained a second button and an inline confirmation panel; Enter still only ever records, and
spending money takes a deliberate click.

**`lib/validateTrade.ts`** extracts the trade rules that were inline in
`app/api/positions/route.ts`, so recording a trade and placing one cannot disagree about what a
valid trade is.

## Security

The app has no login and is tunnel-exposed, so the auth gateway is the only thing between the
internet and a leveraged position. Defence in depth: `ORDER_LIVE_ENABLED` defaults false,
`/api/orders` requires `ORDER_API_TOKEN`, and the browser is given only a 30-minute token minted
from it - the secret never reaches the page (verified: it does not appear in the rendered HTML).
Before any mainnet key: futures only, withdrawals off, spot off, host IP whitelisted.

## Non-obvious findings

- BTCUSDT futures `stepSize` is `0.001` on mainnet but `0.0005` on testnet, and `MIN_NOTIONAL`
  is 50 USDT. Precision must be read from `exchangeInfo` on the network you are on.
- `FloorToStep` needs an epsilon: `0.003/0.001` is `2.9999999999999996` in binary float.
- Binance `-4046` on `marginType` is the requested state reported as an error.
- A failure has to put its reason in the stdout JSON, not only stderr - the caller reads stdout,
  and "exit 1" names no setting to fix. Found during the live e2e and fixed.
- `scripts/watch.ts` hardcoded `/home/tuan/personal/trace` as the capture cwd; now derived from
  the file's own location so a future move cannot break cron silently.

## Verification

- 27 Go tests (`npm run test:go`): the documented Binance HMAC vector, network URL split, lot
  rounding, cap rejection with zero HTTP requests asserted against a recording stub, `-4046`
  tolerance, dry run withholding every POST, and the entry-filled-stop-failed path.
- 136 node tests (`npm test`), tsc clean, eslint 0 errors, production build passes.
- Live against real Binance testnet and mainnet public endpoints: dry runs priced and sized
  correctly on both, guardrails refused a below-minimum order and an over-cap order.
- Full HTTP e2e against a dev server on an isolated copy: 401 without/with a wrong token, 400 on
  a wrong-side stop, on an over-cap notional, on a missing stop and on a tampered preview token,
  200 with correct numbers on a valid preview, 403 everywhere with live ordering off, and the
  page rendering the disabled button with the reason.

## Left to do

1. **Update the crontab** - three entries still point at `/home/tuan/personal/trace`.
2. Create testnet futures keys and place one real testnet order end to end.
3. Only then: restricted mainnet key, `BINANCE_TESTNET=false`, low cap, one minimum-size order.
