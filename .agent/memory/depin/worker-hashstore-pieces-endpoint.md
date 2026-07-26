---
type: decision
tags: [depin, worker, hashstore, debug-endpoint, piecestore]
created: 2026-07-13
agent: claude (main thread)
---

Added a read-only worker debug endpoint `GET /hashstore-pieces/` that lists the
pieces a worker holds in its hashstore. Worker-side counterpart to coord's
`/segment-holders/` ([[project_segment_holders_debug]] equivalent). Brainstormed
+ specced first (spec: `projects/depin/specs/2026-07-13-worker-hashstore-pieces-endpoint.md`).

## API
`GET /hashstore-pieces/?coord=<nodeID>&limit=<n>&cursor=<hex piece_id>` on the
worker debug server (loopback). `coord` optional (one coordinator DB; omit =
all), `limit` default 1000 cap 10000, `cursor` = last piece_id to resume after.
Returns `{coord_filter, scanned_coords[], count, next_cursor, pieces:[{piece_id,
coord, size, created, expires, trash, log}]}` (indented JSON). Errors: bad
coord/cursor/limit -> 400; out-of-range limit clamps; empty -> `pieces:[]`.

## Implementation (all on branch develop, uncommitted)
- `worker/pkg/hashstore/store.go` + `db.go`: added exported `Store.Range` and
  `DB.Range` (thin wrappers over the internal `Tbl.Range`; upstream Storj has no
  higher-level enumeration either). `DB.Range` scans active then passive.
- `worker/piecestore/backend.go`: added `HashStoreBackend.RangePieces(ctx,
  *coord, fn)` iterating open coord DBs (from `dbsCopy()`) in sorted order.
- `worker/hashstore_pieces.go` (new): the `debug.Extension`
  (`hashstorePiecesExtension`), mirroring `coord/segment_holders.go`.
- `worker/peer.go` (~line 318): pass the extension into `debug.NewServer`.
  **Gotcha:** the debug server is built BEFORE the hashstore backend (peer.go
  :318 vs :417), so the extension resolves the backend via a lazy closure
  `func() *piecestore.HashStoreBackend { return peer.Storage.HashStoreBackend }`
  (nil until startup completes -> Handler returns 503 if called too early).

## Key facts / gotchas
- Only enumerator is `Tbl.Range(ctx, func(ctx, Record)(bool,error))`; bool =
  CONTINUE (false stops, err propagates). `rec` is REUSED across slots, so the
  handler copies `rec.Key` before retaining it.
- No key-sorted index (Range is hash-slot order), so pagination is done in the
  handler: a size-`limit` MAX-heap keyed by piece_id keeps the `limit` smallest
  ids > cursor over one O(N) scan. O(limit) memory, resumable, key-ordered.
- **`vo.PieceID.String()` is BASE32, not hex** (`vo.PieceID []byte`,
  `vo.UUID []byte`). Base32 ASCII order != raw-byte order (digits sort before
  letters in ASCII but encode higher values), so order by `bytes.Compare` on the
  raw key, and tests must compare decoded bytes, not the base32 string. This bit
  the first test run.
- `Record` per-piece: Key(32b)/Offset/Log/Length/Created(day)/Expires(day+trash).
  `size` reported = raw `Record.Length` (INCLUDES the 512-byte piece footer).
  `hashstore.DateToTime(days)` converts the day counters.
- Test setup: `piecestore.NewHashStoreBackend(ctx,
  hashstore.CreateDefaultConfig(hashstore.TableKind_HashTbl, false), tmp, "",
  bfm, rtm, nil)`, write via `backend.Writer(ctx, coord, pid, HashAlgorithmSHA256,
  time.Time{})` + `wr.Commit(ctx, &piecepb.PieceHeader{Hash: wr.Hash()})`.

## Verification
`worker/hashstore_pieces_test.go`: unit (two-coord pagination completeness +
strict raw-byte order + coord filter + 400/empty/503 edge cases) AND a LIVE e2e
(`TestHashStorePiecesServedByDebugServer`) that runs a real `debug.NewServer`
with the extension on a real TCP socket and HTTP-GETs it. All pass; full
`worker/pkg/hashstore` suite green (135s); `cmd/worker` builds. Not committed
(no-auto-commit). Dev reach: `dev/spawn-workers.sh` now pins a deterministic
debug port (tcp N -> 32000+N) shown in `info`, so
`curl http://127.0.0.1:32001/hashstore-pieces/`.
