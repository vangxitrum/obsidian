---
type: reference
tags: [development, tunnel, expo, api]
created: 2026-07-23
agent: main
---

Development uses persistent sish-manager tunnels:

- `https://rml-api.tunnel.appdemo.cyou` -> `localhost:18080`
- `https://rml-app.tunnel.appdemo.cyou` -> `localhost:19000`

The Expo client stores the API endpoint in ignored `apps/mobile/.env.local`. Web CORS allows the public app tunnel plus the local Metro origins.

The local runtime uses tmux session `love-reminder`: the Go API is in window `api` on `http://localhost:18080`, the Next.js console is in window `console` on `http://localhost:3003`, and Expo Metro is in window `expo` on port `19000` through the public app tunnel. The backend and console ignored configuration files are `services/backend/.env.local` and `apps/web/.env.local`; never copy their publishing token, reviewer password, or session secret into memory. PostgreSQL runs through Compose on port `5441`. As of 2026-08-20, the notification worker is not running in this recreated session.

The project uses Expo SDK 54 to support the Google Play Expo Go 54.0.8 client. Start Metro behind the custom HTTPS tunnel with `EXPO_PACKAGER_PROXY_URL=https://rml-app.tunnel.appdemo.cyou npx expo start --port 19000`; without the proxy override, Expo generates bundle URLs containing unreachable port `19000`.

Keep the `api` and `expo` tmux windows shell-owned and run the long-lived commands inside them. This allows `Ctrl-C` rebuilds without destroying the window; use Expo's `--clear` flag when a full client rebuild is required.

Expo Go on Android can leave `expo-font` runtime loading pending even after Metro serves the bundle and font assets. The root layout bounds the font-controlled splash gate to five seconds so a stalled font loader cannot trap the app on its native splash indefinitely.
