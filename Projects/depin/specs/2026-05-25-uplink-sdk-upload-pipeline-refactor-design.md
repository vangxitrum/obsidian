# uplink-sdk upload pipeline refactor

Decompose the monolithic `upload.go` into a layered pipeline modeled on
storj uplink's `private/storage/streams/` architecture: splitter,
segment upload, piece upload manager, stream batcher, and scheduler.

## Goals

- Each upload concern lives in its own `internal/` package with a clear
  interface.
- The public `UploadFile` method becomes a thin driver that wires the
  pipeline together.
- Future capabilities (retry, stall detection, limit exchange, batched
  RPCs) can be added per-package without touching the rest of the
  pipeline.
- Existing behavior is preserved: encryption, inline vs erasure path,
  fan-out with optimal-threshold cancellation, CommitSegment/CommitFile.
- Support a CLONE redundancy scheme (full replication) alongside
  REED_SOLOMON: each worker receives the entire segment as-is, no
  erasure coding. The segment IS the piece.

## Non-goals

- Retry on piece failure (limit exchange with coord).
- Stall detection / long-tail timeout.
- Multi-segment concurrency (scheduler is introduced but capped at 1
  for now).
- Batch-aggregating multiple BeginSegment RPCs into a single round-trip.
- Download path refactor.

## Package layout

```
uplink-sdk/
├── client.go                         # unchanged
├── options.go                        # unchanged
├── upload.go                         # simplified driver
├── internal/
│   ├── dial/                         # existing
│   ├── splitter/
│   │   └── splitter.go               # splits data → Segment values
│   ├── streambatcher/
│   │   └── batcher.go                # wraps coord Begin/Commit RPCs
│   ├── segmentupload/
│   │   └── segment.go                # one-segment lifecycle
│   ├── pieceupload/
│   │   ├── manager.go                # fan-out, result collection, cancellation
│   │   └── upload.go                 # single piece upload
│   └── scheduler/
│       └── scheduler.go              # concurrency limiter
```

## Data flow

```
UploadFile (upload.go)
  │
  ├─ CreateFile RPC
  │
  ├─ Splitter.Next() ──────────────────────────┐
  │    splits io.Reader into Segment values     │
  │    each Segment is raw plaintext            │
  │                                             │
  │  ┌──────────────────────────────────────────┘
  │  │
  ├─ Scheduler.Acquire()                        (concurrency gate)
  │
  ├─ SegmentUpload.Begin()
  │    │
  │    ├─ StreamBatcher.BeginSegment()          (coord RPC)
  │    │    returns OrderLimits, RedundancyScheme
  │    │
  │    ├─ if inline → StreamBatcher.MakeInlineSegment()
  │    │
  │    └─ if remote:
  │         ├─ eestream.EncodeReader2()          (erasure encode)
  │         └─ PieceUploadManager.UploadAll()
  │              │
  │              ├─ goroutine per OrderLimit:
  │              │    WorkerClient.PutPiece(limit, pieceReader)
  │              │
  │              ├─ collect results
  │              ├─ cancel remaining at OptimalThreshold
  │              └─ fail if < RequiredCount
  │
  ├─ SegmentUpload.Wait()
  │    └─ StreamBatcher.CommitSegment()         (coord RPC)
  │
  ├─ Scheduler.Release()
  │
  └─ CommitFile RPC
```

## Section 1 — Splitter

### Package

`uplink-sdk/internal/splitter/`

### Types

```go
package splitter

import (
    "context"
    "io"
)

// EncrypterFactory creates a per-segment encrypter.
// partNum is 1-indexed.
type EncrypterFactory func(partNum int32, startingNonce []byte, derivedKey []byte) (EncryptTransformer, error)

// EncryptTransformer wraps a reader with encryption + padding.
type EncryptTransformer interface {
    Transform(r io.Reader) io.ReadCloser
    InBlockSize() int
}

// Segment is one chunk of the file, ready for upload.
type Segment struct {
    Number          int32
    PlainSize       int64
    Reader          io.ReadCloser   // encrypted + padded; nil when IsInline
    IsInline        bool
    EncryptedInline []byte          // populated only when IsInline
}

type Splitter struct {
    reader          io.Reader
    totalSize       int64
    segmentSize     int64
    inlineThreshold int64
    remaining       int64
    nextNum         int32
}

// New creates a Splitter that produces segments from reader.
func New(reader io.Reader, totalSize, segmentSize, inlineThreshold int64) *Splitter

// Next returns the next Segment or io.EOF.
// The caller must close Segment.Reader (if non-nil) before calling Next again.
func (s *Splitter) Next() (*Segment, error)
```

### Behavior

- Splits the input `io.Reader` into segments of at most `segmentSize`
  bytes.
- The last segment may be smaller.
- `Number` is 1-indexed to match the existing partNum convention.
- For the initial version, the splitter does NOT apply encryption — it
  returns raw data readers. Encryption is applied in `segmentupload`
  where the `BeginSegmentResponse` provides the nonce and derived key.
  The `EncrypterFactory` / `EncryptTransformer` types are defined here
  as the interface contract but not used by the splitter directly.
- Inline detection is deferred to `segmentupload` as well (requires
  knowing `inlineThreshold` after `BeginSegment`). The splitter stores
  `inlineThreshold` for use by callers but does not read-ahead.

## Section 2 — StreamBatcher

### Package

`uplink-sdk/internal/streambatcher/`

### Types

```go
package streambatcher

import (
    "context"

    "aioz-depin/internal/vo"
    filepb "aioz-depin/pkg/pb/coord/file/v1"

    "go.uber.org/zap"
)

// Batcher wraps coord file RPCs. Initially pass-through; the interface
// enables future request batching without changing callers.
type Batcher struct {
    file   filepb.FileServiceClient
    logger *zap.Logger
}

func New(file filepb.FileServiceClient, logger *zap.Logger) *Batcher

func (b *Batcher) BeginSegment(ctx context.Context, fileID vo.UUID, segNum int32) (*filepb.BeginSegmentResponse, error)

func (b *Batcher) MakeInlineSegment(ctx context.Context, req *filepb.MakeInlineSegmentRequest) error

func (b *Batcher) CommitSegment(ctx context.Context, req *filepb.CommitSegmentRequest) error
```

### Behavior

- Each method is a thin wrapper around the corresponding
  `FileServiceClient` call.
- Logs at debug level on entry and error.
- Future: batch multiple `BeginSegment` calls into a single coord
  round-trip (like storj's `batchaggregator`).

## Section 3 — PieceUpload Manager

### Package

`uplink-sdk/internal/pieceupload/`

### Types

```go
package pieceupload

import (
    "context"
    "io"

    sharedpb "aioz-depin/pkg/pb/shared"
    piecestorepb "aioz-depin/pkg/pb/worker/piecestore/v1"
    ers "aioz-depin/pkg/ers_schema"

    "go.uber.org/zap"
)

// WorkerClient uploads a single piece to a worker.
type WorkerClient interface {
    PutPiece(ctx context.Context, limit sharedpb.OrderLimit, data io.ReadCloser) (*piecestorepb.PieceHash, error)
}

// Result is one successful piece upload.
type Result struct {
    PieceNum   int32
    Hash       *piecestorepb.PieceHash
    OperatorID []byte
}

// Manager fans out piece uploads and collects results.
type Manager struct {
    wc     WorkerClient
    rs     ers.RedundancyStrategy
    logger *zap.Logger
}

func NewManager(wc WorkerClient, rs ers.RedundancyStrategy, logger *zap.Logger) *Manager

// UploadAll uploads pieces to workers in parallel.
// It cancels remaining uploads once OptimalThreshold is reached.
// Returns error if fewer than RequiredCount pieces succeed.
func (m *Manager) UploadAll(
    ctx context.Context,
    orderLimits []sharedpb.OrderLimit,
    pieceReaders []io.ReadCloser,
) ([]Result, error)
```

### `upload.go` — single piece helper

```go
package pieceupload

// UploadOne uploads a single piece and returns the result.
// Called by Manager goroutines.
func UploadOne(
    ctx context.Context,
    wc WorkerClient,
    pieceNum int32,
    limit sharedpb.OrderLimit,
    data io.ReadCloser,
) (*Result, error)
```

### Behavior

- `UploadAll` spawns one goroutine per order limit.
- Each goroutine calls `UploadOne` which delegates to
  `WorkerClient.PutPiece`.
- Results are collected on a buffered channel.
- When `successes >= OptimalThreshold`, cancel the context shared by
  remaining goroutines.
- If `successes < RequiredCount` after all goroutines finish, return an
  aggregate error.
- If `successes < RepairThreshold`, log a warning but succeed.
- This is a direct extraction of the current `putSegmentPieces` fan-out
  logic into a dedicated package.

## Section 4 — SegmentUpload

### Package

`uplink-sdk/internal/segmentupload/`

### Types

```go
package segmentupload

import (
    "context"
    "io"

    "aioz-depin/internal/vo"
    "aioz-depin/pkg/common"
    "aioz-depin/pkg/encryption"
    eestream "aioz-depin/pkg/eestream"
    ers "aioz-depin/pkg/ers_schema"
    filepb "aioz-depin/pkg/pb/coord/file/v1"
    "aioz-depin/uplink-sdk/internal/pieceupload"
    "aioz-depin/uplink-sdk/internal/splitter"
    "aioz-depin/uplink-sdk/internal/streambatcher"

    "go.uber.org/zap"
)

// Upload manages the lifecycle of one segment.
type Upload struct {
    seg        *splitter.Segment
    batcher    *streambatcher.Batcher
    wc         pieceupload.WorkerClient
    fileID     vo.UUID
    contractID vo.UUID
    encPars    common.EncryptionParameters
    logger     *zap.Logger

    // populated by Begin
    beginResp    *filepb.BeginSegmentResponse
    pieceMgr     *pieceupload.Manager  // created from BeginSegment response
    pieceReaders []io.ReadCloser
    results      []pieceupload.Result
}

// Begin calls BeginSegment on coord, encrypts the segment data,
// builds the RedundancyStrategy and PieceUploadManager, and
// prepares piece readers (erasure-coded or cloned).
func Begin(
    ctx context.Context,
    batcher *streambatcher.Batcher,
    wc pieceupload.WorkerClient,
    seg *splitter.Segment,
    fileID, contractID vo.UUID,
    encPars common.EncryptionParameters,
    inlineThreshold int64,
    logger *zap.Logger,
) (*Upload, error)

// Wait blocks until all pieces are uploaded, then commits the segment.
func (u *Upload) Wait(ctx context.Context) error
```

### Behavior

`Begin`:

1. Call `batcher.BeginSegment(fileID, seg.Number)`.
2. Derive nonce from `beginResp.Nonce` incremented by `seg.Number`.
3. Create encrypter from `beginResp.DerivedKey` + nonce + `encPars`.
4. Wrap `seg.Reader` with `PadReader` → `TransformReader`.
5. **Inline path**: if `seg.PlainSize <= inlineThreshold`, read all
   encrypted bytes, call `batcher.MakeInlineSegment`, return. `Wait`
   becomes a no-op.
6. **Remote path**: build `RedundancyStrategy` from `beginResp`,
   pad for erasure, call `eestream.EncodeReader2` to get piece readers.

`Wait`:

1. Call `pieceMgr.UploadAll(orderLimits, pieceReaders)`.
2. Compute `encryptedSize` via `encryption.CalcEncryptedSize`.
3. Build `CommitSegmentRequest` with results.
4. Call `batcher.CommitSegment`.

## Section 5 — Scheduler

### Package

`uplink-sdk/internal/scheduler/`

### Types

```go
package scheduler

import "context"

// Scheduler limits how many segments upload concurrently.
type Scheduler struct {
    sem chan struct{}
}

// New creates a scheduler that allows up to maxConcurrent segments.
func New(maxConcurrent int) *Scheduler

// Acquire blocks until a slot is available or ctx is cancelled.
func (s *Scheduler) Acquire(ctx context.Context) error

// Release returns a slot to the pool.
func (s *Scheduler) Release()
```

### Behavior

- Backed by a buffered channel of size `maxConcurrent`.
- `Acquire` sends to the channel (blocks when full).
- `Release` receives from the channel.
- For this initial version, `maxConcurrent = 1` (sequential segment
  uploads, matching current behavior). Callers can increase it later.

## Section 6 — Revised upload.go

The public `upload.go` becomes a thin driver:

```go
func (c *Client) UploadFile(ctx context.Context, r io.Reader, p UploadParams) (vo.UUID, error) {
    // Validate inputs
    // CreateFile RPC → FileInfo

    // Create pipeline components
    split := splitter.New(r, p.Size, fileInfo.SegmentSize, fileInfo.InlineThreshold)
    batcher := streambatcher.New(c.file, c.logger)
    sched := scheduler.New(1) // sequential for now
    wc := workerclient.NewWorkerClient(c.logger)

    // Process segments
    for {
        seg, err := split.Next()
        if err == io.EOF { break }

        sched.Acquire(ctx)

        // segmentupload.Begin internally calls BeginSegment, builds the
        // RedundancyStrategy from the response, and creates a
        // pieceupload.Manager scoped to this segment's order limits.
        upload, err := segmentupload.Begin(ctx, batcher, wc, seg,
            fileID, contractID, encPars, inlineThreshold, c.logger)
        if err != nil { return err }

        if err := upload.Wait(ctx); err != nil { return err }
        sched.Release()
    }

    // CommitFile RPC
}
```

The `RedundancyStrategy` and `PieceUploadManager` are both built
inside `segmentupload.Begin` since they depend on the
`BeginSegmentResponse` (order limits and redundancy scheme).

## Section 7 — WorkerClient interface

The existing `pkg/workerclient` continues to provide the concrete
implementation. The new `pieceupload.WorkerClient` interface is
satisfied by `pkg/workerclient.WorkerClient` without changes:

```go
// pkg/workerclient already implements:
type WorkerClient interface {
    PutPiece(ctx context.Context, limit sharedpb.OrderLimit, data io.ReadCloser) (*piecestorepb.PieceHash, error)
}
```

No changes to `pkg/workerclient` are needed. The `pieceupload` package
depends only on the interface, not the concrete type.

## Section 8 — CLONE redundancy scheme

### Proto change

Add `CLONE = 2` to the `Algorithm` enum in
`pkg/pb/shared/redundancy.proto`:

```proto
enum Algorithm {
  ALGORITHM_UNSPECIFIED = 0;
  REED_SOLOMON = 1;
  CLONE = 2;
}
```

Regenerate Go code.

### Semantics

When `RedundancyScheme.Algorithm == CLONE`:

- `RequiredShares = 1` (any single copy can reconstruct the file).
- `RepairShares`, `OptimalShares`, `TotalShares` = N (number of
  workers to replicate to, set by coord).
- `ShareSize` is ignored (each piece is the full segment).

The segment **is** the piece. No erasure encoding is applied.

### Impact on SegmentUpload

`segmentupload.Begin` checks the algorithm from
`BeginSegmentResponse.RedundancyScheme`:

- **REED_SOLOMON** (existing path): pad for erasure →
  `eestream.EncodeReader2` → N piece readers, each a different
  erasure-coded share.
- **CLONE** (new path): skip erasure encoding entirely. Create N
  readers that all read from the same encrypted data. Since each
  goroutine needs its own reader, the encrypted segment data is
  buffered into memory (or a temp file for large segments) and each
  piece reader wraps a `bytes.NewReader` over the same buffer.

```go
if scheme.Algorithm == sharedpb.Algorithm_CLONE {
    buf, err := io.ReadAll(encryptedReader)
    // ...
    for i := range orderLimits {
        pieceReaders[i] = io.NopCloser(bytes.NewReader(buf))
    }
} else {
    // existing Reed-Solomon path
    pieceReaders, err = eestream.EncodeReader2(ctx, padded, rs)
}
```

### Impact on PieceUpload Manager

No changes. The manager already fans out one goroutine per order limit
with one reader per piece. It does not care whether the readers contain
erasure shares or full clones.

### Impact on Splitter

No changes. The splitter splits raw data into segments; redundancy is
applied downstream.

### Impact on ers_schema

`NewRedundancyStrategyFromSchema` currently assumes Reed-Solomon. For
CLONE, `RequiredCount()` returns 1, `OptimalThreshold()` and
`RepairThreshold()` return N. A CLONE strategy with
`RequiredShares=1` works correctly with the existing threshold logic
in the piece manager (cancel remaining at optimal, fail below required).

If `ers_schema` validates that `RequiredShares >= 1` and
`TotalShares >= RequiredShares`, CLONE parameters pass validation as-is.
No changes needed unless validation is stricter.

## Section 9 — Tests

### Unit tests per package

Each new package gets its own `_test.go`:

- `splitter/splitter_test.go` — verify segment splitting with various
  file sizes (exact multiple of segment size, remainder, single segment,
  inline-sized).
- `pieceupload/manager_test.go` — mock `WorkerClient`, verify
  optimal-threshold cancellation, required-count failure, result
  collection.
- `scheduler/scheduler_test.go` — verify acquire/release, context
  cancellation.
- `streambatcher/batcher_test.go` — mock `FileServiceClient`, verify
  pass-through behavior.
- `segmentupload/segment_test.go` — mock batcher + piece manager,
  verify inline vs remote paths, and CLONE vs REED_SOLOMON branching.

### Integration tests

Existing `uplink-sdk/test/integration_test.go` tests continue to work
unchanged — `UploadFile` still presents the same public API.

## Implementation order

1. **Proto: CLONE algorithm** — add `CLONE = 2` to the `Algorithm`
   enum, regenerate.
2. **Splitter** — pure data splitting, no external deps.
3. **Scheduler** — channel-based semaphore, trivial.
4. **PieceUpload Manager** — extract fan-out from `putSegmentPieces`.
5. **StreamBatcher** — thin coord RPC wrapper.
6. **SegmentUpload** — ties encryption + erasure + piece manager
   together; includes CLONE vs REED_SOLOMON branching.
7. **Revised upload.go** — rewire to use the pipeline.
8. **Tests** — unit tests per package, verify integration tests pass.

## Migration strategy

- All new packages are `internal/`, so no public API changes.
- `UploadFile` signature and behavior are identical.
- `pkg/workerclient` is untouched.
- The refactor is purely internal; downstream callers (integration tests,
  `cmd/upload-demo`) require zero changes.
