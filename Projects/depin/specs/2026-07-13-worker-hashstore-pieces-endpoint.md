---
type: spec
project: depin
created: 2026-07-13
status: implemented
tags: [depin, worker, hashstore, debug-endpoint, piecestore]
---

# Worker hashstore-pieces debug endpoint

## Context

A worker stores the pieces it holds in a **hashstore** backend
(`worker/pkg/hashstore`, one `hashstore.DB` per coordinator node-id). Today
there is no way to see which pieces a worker actually holds without reading the
raw log/table files by hand. The coordinator already has a read-only debug
counterpart, `/segment-holders/` (`coord/segment_holders.go`), that traces which
workers hold a segment; this adds the worker-side view: "what pieces does THIS
worker have on disk."

Goal: a local, read-only HTTP endpoint on the worker's existing debug server
that lists the pieces stored in the hashstore, filterable by coordinator and
safely paginated so it stays bounded on large stores.

Non-goals: mutating anything; exposing piece *contents*; a public/gRPC surface;
per-piece footer metadata (order limit, hash) - see "Rejected" below.

## API

```
GET /hashstore-pieces/?coord=<nodeID>&limit=<n>&cursor=<hex-piece_id>
```
- `coord` (optional): coordinator node-id (UUID string) whose hashstore DB to
  scan. Omitted = scan every coord DB the worker holds.
- `limit` (optional): max pieces in the page. Default 1000, hard cap 10000.
- `cursor` (optional): hex `piece_id`; the page returns pieces strictly greater
  than it (keyset resume). Omit for the first page.

Response (indented JSON, `Content-Type: application/json`):
```json
{
  "coord_filter": "<nodeID or 'all'>",
  "scanned_coords": ["<nodeID>", "..."],
  "count": 1000,
  "next_cursor": "<hex piece_id, empty when exhausted>",
  "pieces": [
    {
      "piece_id": "<hex>",
      "coord": "<nodeID>",
      "size": 2319,
      "created": "2026-07-11",
      "expires": null,
      "trash": false,
      "log": 42
    }
  ]
}
```

Field notes:
- `size` = hash-table `Record.Length` (raw bytes stored for the piece; this
  INCLUDES the 512-byte piece footer). Documented as-is; not adjusted.
- `created` = `DateToTime(Record.Created)` formatted `YYYY-MM-DD` (day
  resolution; that is all the Record stores).
- `expires` = `Record.Expires.Time()` (RFC3339) or `null` when no TTL.
- `trash` = `Record.Expires.Trash()`.
- `log` = `Record.Log` (which log file holds the piece).

Errors (mirror `coord/segment_holders.go`): invalid `coord` -> 400; invalid
`cursor` hex -> 400; `limit` non-numeric -> 400; numeric `limit` out of range is
clamped to `[1, 10000]` (not an error). Empty store or no match -> 200 with
`"pieces": []` and empty `next_cursor`.

## Design

Three units.

### 1. Enumeration passthrough (`worker/pkg/hashstore`)

The only record iterator is the internal `Tbl.Range` (`hashtbl.go:296`,
`memtbl.go`), reachable only inside the package - same as upstream Storj (which
has no higher-level enumeration either). Add thin exported wrappers, minimal
divergence from Storj:
- `func (s *Store) Range(ctx, fn func(ctx, Record) (bool, error)) error` -> calls
  `s.tbl.Range`.
- `func (d *DB) Range(ctx, fn) error` -> ranges the `active` then `passive`
  store, **deduping by key** (a key can appear in both during compaction; the
  record from `active` wins). Skips trash records unless a future flag says
  otherwise (v1: include trash, mark `trash:true`).

### 2. Backend accessor (`worker/piecestore/backend.go`)

On `HashStoreBackend` add:
- `func (hsb *HashStoreBackend) Coords(ctx) ([]vo.UUID, error)` - coord ids from
  open `hsb.dbs` plus on-disk `<logsPath>/<coordID>/` dirs, reusing the existing
  `os.ReadDir(logsPath)` + `vo.NewIDFromString` pattern (`backend.go:107-121`).
- `func (hsb *HashStoreBackend) RangePieces(ctx, coord *vo.UUID, fn func(coordID vo.UUID, rec hashstore.Record) (bool, error)) error` -
  for the filtered coord (or every coord from `Coords`), `getDB` then `DB.Range`,
  invoking `fn`. Stops early when `fn` returns `false`.

### 3. Debug extension + wiring

`worker/hashstore_pieces.go`: a `debug.Extension` (`hashstorePiecesExtension`)
with `Description()`, `Path() == "/hashstore-pieces/"`, and `Handler(...)`. Same
shape as `coord/segment_holders.go`. The handler:
1. Parses `coord`, `limit` (default/cap), `cursor`.
2. Runs the **bounded key-ordered paging** below over `backend.RangePieces`.
3. Encodes the response with `json.NewEncoder` + `SetIndent("", "  ")`.

Wiring: pass the extension into `debug.NewServer(...)` at `worker/peer.go:318`
(today the worker passes no extensions), constructed from the already-built
`Storage.HashStoreBackend` (`worker/peer.go:119`, :417). Guard with
`if backend != nil`.

### Pagination algorithm (bounded, key-ordered)

The hashstore has no key-sorted index (`Tbl.Range` visits slots in hash order),
so ordering is built in the handler:
- Maintain a bounded max-heap of size `limit` keyed by `piece_id`.
- For each record from `RangePieces` with `piece_id > cursor`: push; if the heap
  exceeds `limit`, pop the max. This keeps the `limit` SMALLEST keys `> cursor`.
- After the scan, drain the heap, sort ascending, emit. `next_cursor` = the last
  (largest) emitted `piece_id`; empty when fewer than `limit` were found.
- Client repeats with `cursor = next_cursor` until `next_cursor` is empty.

Cost: **O(limit) memory**, **O(N) scan per page** (N = pieces in scanned coords).
Correct, stable ordering by `piece_id`, resumable. Acceptable for a debug tool;
large stores use larger `limit` to reduce page count.

## Security

Read-only. Served only on the worker debug server, which binds loopback by
default (`127.0.0.1:0`; dev now pins a deterministic per-worker port via
`spawn-workers.sh`). No auth on that port by design (same trust model as
`/metrics`, pprof, and coord's `/segment-holders/`); never exposed publicly.

## Rejected alternatives

- **Per-piece footer metadata** (order limit, content hash, signature): would
  require opening every piece to read its 512-byte footer (`backend.go:436`).
  Too expensive for a listing; the Record fields suffice.
- **Aggregate stats only**: `HashStoreBackend.Stats()` already covers counts and
  bytes; it does not list individual pieces, which is the point here. Could be a
  separate `/hashstore-stats/` later.
- **Single unbounded dump**: simplest but unbounded memory/response on real
  stores; rejected for safety.
- **Non-resumable single bounded pass** (no cursor): cheaper (O(N) once) but
  cannot walk the whole store; rejected because the user wants full pagination.

## Testing

Unit (`worker/hashstore_pieces_test.go`): open a real `HashStoreBackend` on a
temp dir, write M pieces for one or two coords via the backend `Writer` +
`Commit`, then drive the handler over `httptest`:
- all M pieces appear exactly once when paging with a small `limit` until
  `next_cursor` empties;
- pages are in ascending `piece_id` order, no dupes, no gaps;
- `coord=<id>` narrows to that coord; unknown coord -> empty;
- invalid `coord`/`cursor` -> 400; `limit` clamps to cap;
- empty store -> `"pieces": []`.

End-to-end: `./dev/spawn-workers.sh start tcp 1`, upload a file through the
network so the worker stores pieces, then
`curl 'http://127.0.0.1:32001/hashstore-pieces/?limit=10'` (debug port from
`./dev/spawn-workers.sh info tcp 1`) and confirm the pieces list, then page with
`cursor`.

## Files touched

- `worker/pkg/hashstore/store.go`, `db.go` - add exported `Range`.
- `worker/piecestore/backend.go` - add `Coords`, `RangePieces`.
- `worker/hashstore_pieces.go` (new) - the `debug.Extension` + handler.
- `worker/peer.go` - pass the extension into `debug.NewServer`.
- `worker/hashstore_pieces_test.go` (new) - unit tests.
