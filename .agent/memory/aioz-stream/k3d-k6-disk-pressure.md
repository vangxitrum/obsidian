---
type: fact
tags: [kubernetes, k3d, k6, disk]
created: 2026-08-03
agent: main
---

The user's k6 setup runs in k3d on a remote machine. Runner pods were evicted with `DiskPressure` from Kubernetes node `gen-stagging`, not `k3d-k6net-server-0`; the latter had 98 GB available and was the wrong node to inspect. Inspect and clean disk/container storage on the specific scheduled node. Local `kind-coord` measurements are unrelated. Avoid mutable `grafana/k6:latest` pulls where practical.
