---
type: fact
tags: [storage, pieces, authorization, expiration]
created: 2026-08-26
agent: main
---

`vo.PiecePrivateKey` / `PiecePublicKey` is a plain Ed25519 key pair and has no expiry field or time-based validation. Every coordinator-issued order limit includes a freshly generated per-limit uplink piece key and has a separate `OrderExpiration`, configured as `coord.order.Expiration` (default `24h`), which workers enforce. The persistent stored piece has no TTL for committed objects: upload limits set `PieceExpiration` to zero, which means no expiration; removal happens through GC after deletion.
