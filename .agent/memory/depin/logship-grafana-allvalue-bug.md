---
type: fact
tags: [logship, loki, grafana, monitoring, bug]
created: 2026-07-22
agent: main
---

Loki rejects LogQL queries where every stream-selector matcher is "empty-compatible" (e.g. `=~".*"`), with a 500 parse error: "queries require at least one regexp or equality matcher that does not have an empty-compatible value ... app=~\".*\" does not meet this requirement, but app=~\".+\" will".

`monitoring/grafana/dashboards/logs/logs-explorer.json` (the Logs Explorer dashboard shipped with [[monitor-server-stack]] / [[telemetry-remote-write-push]]'s log-shipping counterpart, `pkg/logship`) had its `app`/`role`/`instance` template variables set to `"allValue": ".*"`. Opening the dashboard fresh (all three default to "All") triggered this Loki error on both panels — read as "no logs, no error" since the per-panel error icon is easy to miss. The actual logship pipeline (client -> Loki -> Caddy bearer auth) was working the whole time; confirmed live via direct Loki API query that relay-role logs were already flowing in.

**Why:** Grafana dashboards elsewhere in this repo (Prometheus/PromQL, see coord-metrics dashboards) freely use `allValue: ".*"` with multi+includeAll variables because PromQL has no such restriction. That pattern does not carry over to LogQL/Loki dashboards.

**How to apply:** fixed by changing `allValue` to `.+` for all three variables (still matches any non-empty label value, which all shipped streams have, but isn't empty-compatible so Loki accepts it). Any future Loki dashboard in this repo with an includeAll template variable must use `.+`, never `.*`.

Also noteworthy: the actually-running monitoring stack (`monitoring-grafana-1`, `monitoring-loki-1`, etc., confirmed via `docker ps`) is bind-mounted from `/home/tuan/work/depin-workspace/depin` (branch `develop`), not from whichever treehouse worktree a session happens to be in. Check `docker inspect <container> --format '{{json .Mounts}}'` before assuming a local edit reaches a running container. Grafana's file provisioner re-scans every 30s (or force via `POST /api/admin/provisioning/dashboards/reload`).
