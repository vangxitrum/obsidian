---
type: fact
tags: [nextjs, dev-server, networking, gotcha]
created: 2026-07-09
agent: main
---

Next.js 15+ dev mode blocks cross-origin requests to dev assets/HMR websocket by default (security guard against DNS rebinding). If the user opens the dev server via anything other than `localhost` — a LAN IP (`http://10.0.0.67:3002`) or a tunnel domain (`https://beauty.tunnel.zvault.ai`, via an SSH reverse tunnel `-R beauty:80:localhost:3002` to `tunnel.zvault.ai`) — the browser gets a fully server-rendered static page but **client-side JS silently fails to load/hydrate**, or the HMR websocket refuses to connect. Every interactive element (buttons, radios, checkboxes) looks present but does nothing, with no obvious error unless you check DevTools Console/Network.

**Why this matters:** spent two debugging cycles on this — first for the LAN IP, then again for the tunnel domain, since each new origin needs to be added separately. Automated Playwright tests against `localhost` always passed and gave false confidence; the real signal was in the browser's own console (a "Cross-origin access to Next.js dev resources is blocked" message, or `NS_ERROR_WEBSOCKET_CONNECTION_REFUSED` for the HMR socket specifically).

**Fix:** add every origin the app will be accessed from to `next.config.ts`:
```ts
const nextConfig: NextConfig = {
  allowedDevOrigins: ["10.0.0.67", "beauty.tunnel.zvault.ai"],
};
```
Restart the dev server after editing (config isn't hot-reloaded). Verified via `curl --http1.1` with `Upgrade: websocket` headers that the HMR socket handshake (`101 Switching Protocols`) succeeds through the tunnel after the fix.

**How to apply:** if a user reports "nothing is clickable" / "can't select anything" on any Next.js dev app in this environment, check what host/origin they're actually loading the page from (LAN IP, tunnel domain, etc.) before assuming a component bug, and make sure that exact origin is in `allowedDevOrigins`. This machine has an SSH tunnel (`ssh -R beauty:80:localhost:PORT -p 2222 tunnel.zvault.ai`) exposing `skin-prod` publicly at `beauty.tunnel.zvault.ai` — note this is a **dev server with no auth in front of it**, and gets scanned by bots within minutes of being reachable (saw probes for `/.env`, `/.git/HEAD`, `/.vscode/sftp.json` in the HMR manifest). Don't leave it running unattended. See [[project-overview]] for skin-prod context.
