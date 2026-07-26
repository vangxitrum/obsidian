---
type: reference
tags: [redis, docker, recovery, runtime]
created: 2026-07-22
agent: main
---

After an unclean host shutdown, the local Compose Redis service can exit with `Bad file format reading the append only file appendonly.aof.1.incr.aof`. The persisted volume is `tam-anh-demo_redis-data`. Recover the valid AOF prefix with `redis-check-aof --fix /data/appendonlydir/appendonly.aof.manifest` in a temporary `redis:7.4.2-alpine` container mounting that volume, then start Redis and restart the API and worker after Redis reports healthy.
