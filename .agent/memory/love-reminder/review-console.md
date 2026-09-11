---
type: decision
tags: [console, nextjs, review, crawler, security]
created: 2026-08-14
agent: main
---

`apps/web` is an internal Next.js 15 review console for crawled merchant data, the browser equivalent of `reviewctl`. Routes: `/review` queue, `/review/[id]` diff and decision, `/sources`, `/runs`. It is the only frontend besides `apps/mobile` (Expo).

**Two secrets, deliberately separate.** `ADMIN_API_TOKEN` authenticates the console's server to the Go API and is a publishing credential; `CONSOLE_PASSWORD` is what a reviewer types. Reusing the API token as the login password would put a publishing credential into a form field and browser history. `lib/api.ts` is marked `server-only`, so importing it from a client component is a build error - that is what guarantees the token never reaches a browser. The session cookie carries the reviewer name plus an HMAC signed with `CONSOLE_SESSION_SECRET`, so decisions cannot be misattributed by editing it. `middleware.ts` only checks cookie presence (edge runtime has no `node:crypto`); the signature is verified in the console layout before any fetch.

**The queue sorts into approval order** (merchant → location/contact → item → offer), because the backend refuses out-of-order approval with a 409 and working top to bottom avoids it.

**Three real bugs were found by building it**, all now fixed and covered by tests:
- `applyMerchant` had `ON CONFLICT (canonical_domain)` but `merchants.slug` is separately unique, so two merchants sharing a name on different domains crashed approval with a raw `merchants_slug_key` error. Now resolved via `uniqueMerchantSlug` (name slug → domain slug → domain-hash suffix).
- A server action re-renders the page, so a successful approval destroyed its own confirmation and showed "already approved" instead of "Created merchant X". `DecisionPanel` now owns the already-decided case.
- Compose's `${VAR:?}` syntax is interpolated for every service even when starting only one, which broke `docker compose up -d postgres`. Console vars use plain substitution and the app validates them itself.

**Why:** Reviewing in a terminal does not scale past a handful of candidates, and a diff is much easier to judge visually than as JSON.

**How to apply:** Playwright tests in `apps/web/e2e` drive a real browser against a live API and database; `e2e/global-setup.ts` reseeds crawler fixtures via `services/crawler/scripts/seed_demo_candidates.py` because approving consumes candidates. Related: [[merchant-crawler-implementation]].

On 2026-08-20 the console readability was revised after browser checks at
1440px and 390px. Keep the 16px base type, stronger muted-text contrast,
prominent filter panel, proposal count, and price/availability summaries for
offers. Dense tables become labeled cards below 700px so confidence, age, and
review actions are never hidden behind horizontal scrolling. When a source is
already selected, omit the repetitive source column. `allowedDevOrigins` and
disabled development indicators keep the local/tunneled review surface free of
Next.js warning controls that obscure content.
