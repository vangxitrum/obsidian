# Trade Journal - UI + backend for M5 plan snapshots

## Context

The M5 script `ROI Entry Planner` (id `e882d94f-2be0-445c-8db9-318dc7002ba3`) computes a leveraged
BTC entry plan (entry / TP / SL / liq, R:R, loss ROI, verdict) and draws it on the chart, but the
values live only in the browser. A companion script `ROI Plan Readback`
(id `7580779b-04ff-436a-9f61-bbbd56617523`) already smuggles the same numbers out through
`script_get_diagnostics` by throwing them in an error message - that is the only data-read channel
M5's MCP exposes today.

Right now every reading is thrown away. We want a record: what the plan was at each point in time,
why it was TRADEABLE or REJECT, and what price actually did afterwards. That history is what turns
"the script says no" into evidence about whether the rule set is any good.

Deliverable: a local Next.js app storing snapshots + reasoning + outcomes, a capture script driven
by cron every 4 hours, and a live demo trace running over the next few days.

Decisions already made by the user: new local Next.js app (not an artifact, not inside
essay-manager), built in **`/home/tuan/personal/trace`**; local cron every 4h; snapshots record plan
numbers, market context, a written reason, and outcome tracking.

## Key finding from exploration

The M5 MCP server is plain streamable HTTP - `https://app.mmt.gg/api/mcp` with a bearer token,
configured per-project in `~/.claude.json` under
`projects["/home/tuan/personal/essay-manager"].mcpServers.m5`. `initialize` returns 200 with a
JSON body and no `mcp-session-id` header.

So **cron can talk to M5 directly** - no `claude -p` wrapper, no Claude session required. The
capture script is an ordinary Node process.

Caveat that shapes the whole design: M5 data comes from the user's **browser session**. If no M5
tab is connected, `m5_get_status` returns no sessions and there is nothing to capture. The script
must detect this and record a skip, not crash.

## Conventions to match (from `/home/tuan/personal/essay-manager`)

Mirror the house style rather than inventing a new one:

- Next 16 + React 19, **npm**, TypeScript strict, path alias `@/*` at repo root (no `src/`).
- **No Tailwind, no component library.** Plain CSS in `app/globals.css` with Catppuccin custom
  properties, following `essay-manager/app/globals.css:1-40`.
- Prisma 7 with a **driver adapter** and a global singleton - copy the shape of
  `essay-manager/lib/prisma.ts:1-13`, swapping `@prisma/adapter-pg` for
  `@prisma/adapter-better-sqlite3` (already a known dep in that repo).
- DB url supplied via `prisma.config.ts` (`essay-manager/prisma.config.ts:5-14`), not in the schema.
- Route handlers at `app/api/<resource>/route.ts`, `export async function GET/POST(request: Request)`,
  plain `NextResponse.json`, manual `typeof` validation, a leading `//` comment naming method+path -
  see `essay-manager/app/api/settings/theme/route.ts:1-19`.
- Tests: Node's built-in runner, `*.test.ts` colocated in `lib/`, `node --test 'lib/**/*.test.ts'`.
- Model conventions: `String @id @default(cuid())`, `createdAt`/`updatedAt`, status values as
  `String` with a trailing comment.
- `.env.example` committed and heavily commented; `.env` gitignored.

## Plan

### 1. Scaffold `/home/tuan/personal/trace`

`git init`, `.nvmrc` = 20, Next 16 app router, TypeScript, npm, no Tailwind. Dev port **3010**
(`next dev --webpack -p 3010`) so it never collides with essay-manager.

### 2. Data model - `prisma/schema.prisma` (SQLite)

`model Snapshot`
- identity: `id`, `capturedAt`, `source` (`cron` | `manual`), `symbol`, `timeframe`
- inputs: `leverage`, `targetRoi`, `maxLossRoi`
- plan: `side`, `entry`, `tp`, `sl`, `liq`, `movePct`, `slPct`, `lossRoi`, `rr`, `liqRatio`,
  `daysToTarget`
- context: `atr`, `atrPct`, `swingHigh`, `swingLow`, `rangeHigh`, `rangeLow`, `emaSlope`, `winBars`
- verdict: `verdict`, `gateRisk`, `gateLiq`, `gateRr` (booleans, so the UI can show which gate failed)
- reasoning: `reasoning` (generated at capture), `note` (user-editable, nullable)
- outcome: `outcome` (`PENDING` | `TP` | `SL` | `EXPIRED` | `SKIPPED`), `resolvedAt`,
  `resolvedPrice`, `mfe`, `mae`
- audit: `raw` - the full `PLAN ...` detail string, so a parser bug is always recoverable

`model CaptureRun` - one row per cron firing: `startedAt`, `ok`, `reason` (e.g.
`no browser session`), `snapshotId?`. Without this, "no data for Tuesday" is indistinguishable from
"cron never ran".

### 3. Readback script v2 (M5 side)

Extend `ROI Plan Readback` to also emit `gapHigh` / `gapLow` - the highest high and lowest low over
the last `ceil(4h / timeframe)` bars. Outcome resolution needs the extremes **between** captures;
close-only comparison would miss a TP or SL that was touched and retraced. Everything else in that
script stays as it is.

### 4. Capture pipeline - `scripts/capture.ts`

Run by cron, writes through Prisma directly (no dev server needed):

1. Read `M5_MCP_URL` + `M5_MCP_TOKEN` from `.env` (chmod 600, gitignored, copied once from
   `~/.claude.json`; never hardcoded).
2. Connect with the official `@modelcontextprotocol/sdk` client over
   `StreamableHTTPClientTransport` - it owns the handshake, so we do not hand-roll JSON-RPC and do
   not depend on the stateless behaviour holding.
3. `m5_get_status` -> if no connected session, write a `CaptureRun` with
   `reason: "no browser session"` and exit 0.
4. `chart_add_script(readback id)` -> `script_get_diagnostics(readback id)` -> take
   `diagnostics[0].detail`.
5. Parse with `lib/parsePlan.ts` (pure, unit-tested): `PLAN k=v k=v ...` -> typed object. Reject a
   detail line missing required keys rather than writing partial rows.
6. Resolve open snapshots first (step 5 below), then insert the new `Snapshot` + `CaptureRun`.

Flags: `--dry-run` (parse and print, no write), `--manual` (sets `source=manual`).

### 5. Outcome resolution - `lib/resolve.ts`

For each `PENDING` snapshot, using the new capture's `gapHigh`/`gapLow`:
- long: `gapHigh >= tp` -> `TP`; `gapLow <= sl` -> `SL`; short mirrored
- both touched in one window -> `SL` (pessimistic; we cannot see the order within the gap, and
  assuming the win would flatter the record)
- older than `daysToTarget * 3` with neither touched -> `EXPIRED`
- track running `mfe` / `mae` on every pass, so even unresolved rows show how far they travelled

### 6. Reasoning text - `lib/reasoning.ts`

Rule-based, generated in-process - no LLM call, so cron stays offline-capable and deterministic.
Composes from the gates and the geometry, e.g.:

> REJECT. Stop sits at the swing low 77,255 (4.44% away); at 5x that is 22.2% of margin at risk
> against a 20% target, R:R 0.90. Price is mid-range - entry near 77,400 would make the same target
> a 3R trade.

The `note` field stays free for the user's own words.

### 7. UI

- `/` - snapshot table (time, side, entry, TP/SL, R:R, loss ROI, verdict badge, outcome badge),
  filters for verdict/outcome, and a header strip: total captures, % TRADEABLE, TP/SL hit rate,
  average R:R. Missed captures visible from `CaptureRun`.
- `/snapshots/[id]` - every field grouped (plan / context / gates / outcome), the reasoning, the raw
  detail line, and a form to edit `note` or override `outcome`.
- API: `GET/POST /api/snapshots`, `PATCH /api/snapshots/[id]`, `GET /api/runs`.

### 8. Cron + demo trace

`scripts/capture.sh` (loads nvm, `cd`s to the repo, runs the capture) and a crontab line:

```
0 */4 * * * /home/tuan/personal/trace/scripts/capture.sh >> /home/tuan/personal/trace/logs/capture.log 2>&1
```

Installed only with the user's explicit go-ahead, since it edits their crontab. Then seed snapshot
#1 from the reading already taken (LONG, entry 80,844.1, tp 84,077.9, sl 77,255.4, liq 65,079.5,
R:R 0.901, lossRoi 22.2%, REJECT) and let it run.

**The demo depends on the M5 tab staying open.** Closed browser = skipped captures, recorded as
such. Worth saying plainly up front.

## Verification

1. `npm test` - parser tests against the real captured detail string; resolution tests for
   TP-hit / SL-hit / both-touched / expiry using synthetic gap extremes.
2. `npm run capture -- --dry-run` - prints the parsed object, writes nothing.
3. `npm run capture` twice a few minutes apart - two `Snapshot` rows, two `CaptureRun` rows.
4. Close the M5 tab, run capture again - exits 0 with a `no browser session` run row, no snapshot.
5. `npm run dev` -> `http://localhost:3010` - rows visible, verdict/outcome badges correct, detail
   page renders, note edit persists.
6. Manual resolution check: insert a snapshot with a TP just below the last price, run capture,
   confirm it flips to `TP` with `resolvedAt` set.
7. After cron is live: confirm the next 4h firing appended a row (`logs/capture.log` + the UI).

## Notes / risks

- Nine `ZZ Probe*` scripts and a duplicate `ROI Entry Planner` layer are still in the user's M5
  account from debugging. MCP has no delete tool - cleanup is manual in the M5 UI.
- `math.roundToTick()` crashes the M5 script runtime in this build; the planner already works around
  it with a local `toTick()`. Do not reintroduce it in the readback script.
- The readback script is fatal-by-design (it throws to return data). It must stay off the chart as a
  visible layer, and the drawing planner must never adopt the throw trick.
- The bearer token is a real credential. It goes in `.env` (600, gitignored) and `.env.example`
  carries only a placeholder.

## Status 2026-09-04
Built and running. App at /home/tuan/personal/trace, dev server port 3011 (3010 was taken).
Cron installed: 0 */4 * * * scripts/capture.sh; requires an M5 browser tab open.
First two snapshots captured (LONG btc/usd entry 81197.8, REJECT, R:R 0.82).
13 unit tests pass; typecheck clean.
