---
type: reference
tags: [runtime, docker, node, development]
created: 2026-07-19
agent: main
---

The development host does not have Node/npm/npx on `PATH`. The local application processes run as independent detached Docker containers with host networking: `hospital-api` and `hospital-worker` use `golang:1.25-bookworm`; `hospital-web-node` and `hospital-app-node` use `node:22-bookworm`, UID/GID `1000:1000`, `HOME=/tmp`, and `corepack pnpm`. The repository is mounted at `/workspace`. These services do not depend on a tmux session; inspect output with `docker logs <container>`. As of 2026-07-19, `hospital-web-node` serves its production Next.js build on host port `3123`.

For LAN access, the host address is `10.0.0.67`. Start API with `WEB_ORIGIN=http://10.0.0.67:3000`, web with `NEXT_PUBLIC_API_URL=http://10.0.0.67:8080` and `NEXT_PUBLIC_WS_URL=ws://10.0.0.67:8080/ws`, and Expo with equivalent `EXPO_PUBLIC_*` values plus `REACT_NATIVE_PACKAGER_HOSTNAME=10.0.0.67`. Merely listening externally is insufficient because frontend `localhost` URLs resolve to the client device.

When serving through `ta-web.tunnel.appdemo.cyou`, the web container must run a production build rather than `next dev`; Next 16 blocks cross-origin development resources and leaves the portal on its pre-hydration loading screen. Build with public `NEXT_PUBLIC_API_URL=https://ta-api.tunnel.appdemo.cyou` and `NEXT_PUBLIC_WS_URL=wss://ta-api.tunnel.appdemo.cyou/ws`, then use `next start` on port 3000. API must use `WEB_ORIGIN=https://ta-web.tunnel.appdemo.cyou`. This setup was browser-verified through the public URLs, including staff login.
