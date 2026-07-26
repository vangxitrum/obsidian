---
type: reference
tags: [development, tunnel, expo, api]
created: 2026-07-23
agent: main
---

Development uses persistent sish-manager tunnels:

- `https://rml-api.tunnel.appdemo.cyou` -> `localhost:18080`
- `https://rml-app.tunnel.appdemo.cyou` -> `localhost:19000`

The Expo client stores the API endpoint in ignored `apps/mobile/.env.local`. The local runtime is hosted in tmux session `love-reminder` with `api`, `worker`, and `expo` windows. Web CORS allows the public app tunnel plus the local Metro origins.

The project uses Expo SDK 54 to support the Google Play Expo Go 54.0.8 client. Start Metro behind the custom HTTPS tunnel with `EXPO_PACKAGER_PROXY_URL=https://rml-app.tunnel.appdemo.cyou npx expo start --port 19000`; without the proxy override, Expo generates bundle URLs containing unreachable port `19000`.
