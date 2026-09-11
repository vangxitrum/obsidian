# docker-storage-layout

Workstation Docker facts (recorded 2026-08-28).

- Docker Engine **29.7.2**, installed from apt (`/usr/bin/dockerd`), not snap.
- Storage driver reports `overlayfs` with `driver-type: io.containerd.snapshotter.v1`
  -> the **containerd snapshotter is active**. Image layers therefore live in
  `/var/lib/containerd`, NOT `/var/lib/docker`. `/var/lib/docker` only holds
  volumes, container metadata, network state and buildkit cache.
  Any "move Docker storage" work must relocate **both** roots.
- Disk layout: `/` = `/dev/nvme0n1p4` ext4 183G (78% used, ~39G free);
  `/home` = `/dev/nvme0n1p5` ext4 716G (~497G free). `/home` is the spill target.
- `/etc/docker/daemon.json` already carries `insecure-registries: ["10.0.0.106:5000"]`
  and an `nvidia` runtime entry - edit it with `jq` merge, never overwrite.
- `/etc/containerd/config.toml` is the stock Docker-shipped file with
  `disabled_plugins = ["cri"]` and `root` commented out.
- Migration script written to `~/.local/bin/move-docker-storage.sh`
  (stops docker+containerd, rsync -aHAX both roots, patches both configs,
  renames originals to `*.old`, verifies counts, restarts containers,
  emits a rollback script under `/root/docker-move-backup-<ts>/`).
