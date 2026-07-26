---
type: reference
tags: [expo, tunnel, demo, mobile]
created: 2026-07-17
agent: main
---

Public cross-network demo endpoints:

- Expo Metro: `https://ta-app.tunnel.appdemo.cyou` forwarding local port `8081`
- API and WebSocket: `https://ta-api.tunnel.appdemo.cyou` forwarding local port `8080`
- Staff portal and payment simulator: `https://ta-web.tunnel.appdemo.cyou` forwarding local port `3123`

Start Metro with `EXPO_PACKAGER_PROXY_URL=https://ta-app.tunnel.appdemo.cyou`, `EXPO_PUBLIC_API_URL=https://ta-api.tunnel.appdemo.cyou`, and `EXPO_PUBLIC_WS_URL=wss://ta-api.tunnel.appdemo.cyou/ws`. Set API `WEB_ORIGIN` and `MOMO_REDIRECT_URL` to the public web endpoint and `MOMO_IPN_URL` to the public API endpoint.

These routes use sish reverse SSH forwards on `tunnel.appdemo.cyou:2222`, for example `ssh -N -T -p 2222 -i ~/.ssh/drive-prod -R ta-web:80:localhost:3123 tunnel.appdemo.cyou`. A missing SSH forward produces a TLS internal error because sish has no active route/certificate for that hostname. The local Next.js server must also be listening on port `3123` with the public API and WebSocket environment variables.

Run the tunnel-facing web portal as a production build (`next build`, then `next start --hostname 0.0.0.0 --port 3123`), not `next dev`; the Next 16 development runtime failed to hydrate and remained on “Đang mở cổng vận hành...” behind this setup. API WebSocket origin validation derives its public allowlist entry from `WEB_ORIGIN`; otherwise browser upgrades from `ta-web` receive HTTP 403.

Audit and repair on 2026-07-21 confirmed the web and mobile bundles embed the public API and WebSocket domains. Android and iOS Expo manifests publish `ta-app.tunnel.appdemo.cyou` for both `hostUri` and launch assets. The compressed Android bundle downloads publicly with HTTP 200; an uncompressed request exceeds the tunnel response limit and returns 502, while Expo clients request compressed content. Public web, payment, OTP authentication, API, and authenticated WebSocket upgrade checks pass.
