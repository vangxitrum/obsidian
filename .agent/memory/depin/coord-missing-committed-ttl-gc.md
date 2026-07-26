---
type: decision
tags: [depin, coord, ttl, expiration, gc, expireddeletion, bug, hashstore, data-loss]
created: 2026-07-13
updated: 2026-07-13
agent: claude (main thread)
---

**CRITICAL data-durability bug found + FIXED (2026-07-13).** Root cause is NOT a
"missing committed-TTL GC chore" (my first framing was wrong); it is that the
coordinator stamped every uploaded piece's storage TTL = the **upload deadline**.

## Root cause (verified in code)
`coord/file/endpoint.go` BeginSegment set `pieceExpires := fileContent.ExpiredAt`
where `ExpiredAt = CreateFile time + FileExpiration` (upload window, default 24h;
`coord/file/service.go`). That flowed to `CreatePutOrderLimits` ->
`coord/order/signer.go` `PieceExpiration` -> worker
`pieceBackend.Writer(..., limit.PieceExpiration)` -> hashstore record `Expires`.
depin has **no user/object TTL** (CreateFile has no expiration; committed files
are "permanent until deleted"), so committed pieces wrongly self-deleted ~24h
after upload -> once pieces expired past the repair threshold the object was
unrecoverable. Proven: worker alias-38 piece 63 (file created 07-08) gone; 20 of
its 30 surviving pieces had TTL 07-12 (already expired on 07-13).

## Fix (implemented this session, on develop, uncommitted)
Chosen approach: pieces never expire; GC (bloom-filter Retain) reclaims
abandoned-upload + deleted-file pieces.
- **A. Pieces permanent:** `coord/file/endpoint.go` -> `var pieceExpires time.Time`
  (zero). Worker treats zero as no-TTL (kept forever): confirmed
  `worker/piecestore/verification.go:185` isExpiredAt(zero)=false,
  `worker/pkg/hashstore/store.go:727` stores Expiration(0), compaction never
  expires 0 (`store.go:963`); signed payload omits zero (`pkg/signing/encode.go:32`).
  Repair PUT (`coord/repair/repairer/segments.go:441`) already passed zero for
  committed segments; inline segments mint no limits. Test:
  `coord/order/put_expiration_test.go`.
- **B. `expireddeletion` (deletes expired PENDING uploads) now runs in split
  deploys:** it was wired ONLY in `NewPeer` (`coord run`), so split prod
  (api/core/…) never ran it. Added `peer.setupExpiredDeletion()` to `NewCore`
  (`coord/core.go`) — the singleton-chore home. (The DB delete itself was correct:
  hard delete, `status=PENDING AND created_at<cutoff`, no gorm soft-delete since
  `File.DeletedAt` is a plain `*time.Time`, not `gorm.DeletedAt`.) Also rewrote
  `DeleteExpiredPendingFiles` (`coord/db/file_repo.go`) to hard-delete the pending
  files AND their segments EXPLICITLY via a CTE (mirrors the committed DeleteFile
  CTE) instead of relying on the `fk_files_segments ON DELETE CASCADE` — self-
  contained + robust to FK/model changes. New real-Postgres test
  `TestFileRepositoryDeleteExpiredPendingFiles` (coord/db/file_repo_test.go)
  asserts pending file+segments gone, committed file+segment kept.
- **C. GC enabled + sender wiring fixed:** the GC sender was wired ONLY in
  `NewPeer`, but must be co-located with the bloom observer (in-process
  `NewMemStorage` hand-off) which rides `setupRangedLoop` (runs in the
  `ranged-loop` role). Added `setupGCSender()` (guarded, + `setupDialer()` since
  ranged-loop otherwise has no worker dialer) to `NewRangedLoop`
  (`coord/rangeloop.go`). Flipped `Enabled` default false->true in
  `coord/gc/bloomfilter/config.go` and `coord/gc/sender/config.go`. Bloom filters
  are conservative (false positives only delay reclaim, never delete live pieces).
- **C3 recovery window:** trash-keep grace = `worker/pkg/hashstore/config.go:58`
  `Compaction.ExpiresDays` default **7** (days trash kept before permanent
  reclaim). LEFT at 7 (safe Storj-aligned); set worker
  `storage.hashstore.compaction.expires-days=2` if a 2-day window is really
  wanted. Retain clock-skew tolerance = `worker/retain/runner_service.go:21`
  MaxTimeSkew 72h.

## Verification done
build ./... OK; new put_expiration test PASS; 25 coord packages PASS; full
hashstore suite PASS earlier. NOT unit-tested: the B/C1 role wiring (needs a DB
to construct roles) — faithful mirror of the existing NewPeer pattern,
build-verified. Full e2e (upload -> `/hashstore-pieces/` shows expires:null;
delete -> GC reclaims) needs a running cluster. Not committed (no-auto-commit).
Relates to [[worker-hashstore-pieces-endpoint]], [[project_segment_holders_debug]].
