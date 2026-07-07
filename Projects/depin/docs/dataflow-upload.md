---
type: doc
project: depin
tags: [depin, dataflow, upload, encryption, erasure-coding]
created: 2026-07-05
---

# dataflow: upload — end-to-end (the write path)

How a file becomes durable, distributed, encrypted pieces across the DePIN network. This is a
**cross-cutting** guide: it follows one upload through every process it touches — the `uplink`
service, the coordinator (`coord`), and the storage `worker`s — naming the real packages and
functions at each hop. For the subsystem-local guides see the *Where to go next* links at the bottom.

> **Encryption is server-side.** The `uplink` is a **server-side gateway service**
> ([`uplink/application/service/upload.go`](../uplink/application/service/upload.go)), not an
> end-user library. It holds the encryption key material and performs AES-GCM encryption *on behalf
> of* the user. The coordinator mints the per-segment `DerivedKey` and `Nonce` and hands them to the
> uplink. Workers store only encrypted bytes and never see keys — but the uplink service (and its
> operator) **can** read the data. This is **server-side encryption, not end-to-end / zero-knowledge.**

---

## 1. Summary

An upload is multipart. The uplink registers the file with the coordinator (`CreateFile`), then for
each part (segment) it: stages the plaintext, opens a segment session (`BeginSegment`, which returns
the coordinator-minted key/nonce + signed PUT order limits), **encrypts the segment with AES-GCM
server-side**, and then either stores it **inline** (small) or **distributes** it (large). For
distributed segments the encrypted stream is split — by Reed-Solomon erasure coding (`REED_SOLOMON`)
or full replication (`CLONE`) — into pieces that are uploaded in parallel to workers. Each worker
verifies the coordinator-signed order limit, stores the piece via its `HashStoreBackend`, and returns
its **own signed piece hash**. The uplink forwards those results in `CommitSegment` (or
`MakeInlineSegment`), and a final `CommitFile` flips the coordinator's file from pending to ready.

The two decision points are: **inline vs. distributed** (`partSize <= InlineThreshold`), and
**redundancy algorithm** (`RedundancyScheme.Algorithm` = `REED_SOLOMON` vs `CLONE`).

---

## 2. The whole path

```
   USER                 UPLINK service (server-side gateway)              COORDINATOR (coord)            WORKER(s)
   ────                 ─────────────────────────────────────            ───────────────────            ─────────

  create  ─────────────▶ CreateMultipartUpload
  upload                 hubServer.File().CreateFile ──────────────────▶ file.Endpoint.CreateFile
                                                                          (returns StreamId,
                          ◀───────────────────────────────────────────── SegmentSize,
                                                                          RedundancyScheme,
                                                                          InlineThreshold)

  PUT part ────────────▶ UploadPart(partReader, partNumber, ...)
                         stagePart: write part.plain + checksum
                         hubServer.File().BeginSegment ───────────────▶ file.Endpoint.BeginSegment
                                                                          • selects workers (overlay)
                                                                          • rand DerivedKey + Nonce
                                                                          • order.CreatePutOrderLimits
                                                                            └ signing.SignOrderLimit  (coord-signed)
                                                                          • signs SegmentID blob
                         ◀──────────────────────────────────────────────  returns: DerivedKey, Nonce,
                                                                          RedundancyScheme, OrderLimits[],
                                                                          PiecePrivateKey, SegmentId

                         ┌─ SERVER-SIDE ENCRYPTION (in the uplink) ─┐
                         │ Increment(Nonce, partNumber)             │
                         │ encryption.NewEncrypter(AESGCM,          │
                         │     DerivedKey, Nonce, DefaultBlockSize) │
                         │ PadReader → TransformReader  (AES-GCM)   │
                         └──────────────────────────────────────────┘
                                            │
                  ┌─────────────────────────┴──────────────────────────┐
       partSize ≤ │ InlineThreshold                       partSize > InlineThreshold
       ───────────▼──────────                              ──────────▼───────────────
       read all ciphertext                                 split the encrypted stream:
       MakeInlineSegment ──────▶ file.Endpoint.            ┌─ REED_SOLOMON: eestream.EncodeReaderSimple
       (EncryptedInlineData)     MakeInlineSegment         │     (RS erasure shares, one reader / piece)
                                 (stores bytes in DB)      └─ CLONE: full-replica pieces; uplink builds
                                                                 merkle.Build tree, root committed later
                                                            │
                                                            ▼  PutSegment: parallel piece upload
                                                       for each OrderLimit ──┐
                                                                            │ workerClient.PutPiece
                                                                            ▼
                                                            PieceStoreEndpoint.Upload  (gRPC stream)
                                                            • verifyOrderLimit (PUT): coord sig via
                                                              trust pool + addressee + expiry +
                                                              usedserials replay check (single-use)
                                                            • verifyOrder per frame (uplink sig)
                                                            • pieceBackend.Writer → HashStoreBackend
                                                            • writer.Commit(PieceHeader)
                                                            • signing.SignPieceHash (WORKER-signed)
                                                       ◀──── UploadResponse{ PieceId, PieceHash }
                                                            │ (wait for OptimalThreshold successes)
                                                            ▼
                         CommitSegment(UploadResult[]) ─▶ file.Endpoint.CommitSegment
                                                          • per piece: signing.VerifyPieceHashSignature
                                                            (worker sig vs check-in pubkey)
                                                          • CLONE: store MerkleRoot + ChunkSize (UNSIGNED)
                                                          • service.CommitSegment → persist segment row

  complete ───────────▶ CompletedMultipartUpload
                         hubServer.File().CommitFile ───▶ file.Endpoint.CommitFile
                                                          (PENDING → READY)
```

---

## 3. Glossary

| Term | Meaning | Where |
| ---- | ------- | ----- |
| **segment / part** | one slice of the file; the unit of encryption + redundancy | `upload.go` `UploadPart` |
| **inline segment** | small segment stored as encrypted bytes directly in the coordinator DB | `MakeInlineSegment` |
| **distributed segment** | large segment split into pieces stored on workers | `PutSegment` |
| **`InlineThreshold`** | plaintext-size cutoff for inline vs. distributed | `BeginSegmentResponse` / `Config.InlineThreshold` |
| **`RedundancyScheme`** | how a segment is split: `Algorithm` ∈ {`REED_SOLOMON`, `CLONE`} + share counts | `pkg/pb/shared/redundancy.proto` |
| **piece** | one worker's share (RS) or full replica (CLONE) of a segment | `SegmentPieceUploadResult` |
| **`DerivedKey` / `Nonce`** | per-segment AES-GCM key + nonce, **minted by the coordinator** in `BeginSegment` | `file/endpoint.go` `BeginSegment` |
| **order limit** | coordinator-signed token authorizing one worker to store one piece | `sharedpb.OrderLimit` |
| **`PiecePrivateKey`** | ephemeral key the uplink uses to sign the single-use order per piece | `order.CreatePutOrderLimits` |
| **piece hash** | SHA-256 of the stored piece; signed by the worker, verified by coord | `piecepb.PieceHash` |
| **`SegmentId` blob** | coordinator-signed opaque blob echoed back on commit; carries segment state | `BeginSegmentResponse.SegmentId` |
| **`HashStoreBackend`** | the worker's log-structured piece store | `worker/piecestore` / `worker/pkg/hashstore` |
| **`OptimalThreshold`** | enough successful piece puts to consider the segment durable | `ers_schema.RedundancyStrategy` |

---

## 4. Step-by-step walkthrough

### 4.1 Register the file — `CreateFile`

[`UploadService.CreateMultipartUpload`](../uplink/application/service/upload.go) calls
`hubServer.File().CreateFile`. The coordinator's
[`file.Endpoint.CreateFile`](../coord/file/endpoint.go) returns the durable **`StreamId`** plus the
storage policy the uplink must obey: **`SegmentSize`**, **`RedundancyScheme`**, and
**`InlineThreshold`** (sourced from the coordinator's storage config — see
`file/endpoint.go` ~L112-114). The uplink records a local file entity (`entity.NewFile`) keyed by the
coordinator's `StreamId`.

### 4.2 Stage and open a segment — `BeginSegment`

For each part, [`UploadService.UploadPart`](../uplink/application/service/upload.go) first
**stages** the plaintext to `part.plain` while computing a checksum (`stagePart`, using
`pkg/checksum`), validating the declared part size. It then calls `hubServer.File().BeginSegment`.

[`file.Endpoint.BeginSegment`](../coord/file/endpoint.go) is where the coordinator:

- selects the holding workers (overlay/placement);
- mints **per-segment key material**: `var nonce common.Nonce; rand.Read(nonce[:])` and
  `var derivedKey common.Key; rand.Read(derivedKey[:])` — **the server generates the encryption
  key**, which is the crux of server-side encryption;
- mints the **PUT order limits** via
  [`order.Service.CreatePutOrderLimits`](../coord/order/service.go), each signed with
  [`signing.SignOrderLimit`](../pkg/signing/sign.go) (`CoordinatorSignature`), and the ephemeral
  `PiecePrivateKey`;
- returns `RedundancyScheme`, `OrderLimits[]`, `Nonce`, `DerivedKey`, `PiecePrivateKey`, and a signed
  `SegmentId` blob.

### 4.3 Server-side AES-GCM encryption

Back in `UploadPart`, the uplink derives the starting nonce and builds the cipher:

```go
startingNonce := common.Nonce(beginSegment.Nonce)
encryption.Increment(&startingNonce, partNumber)
encrypter, _ := encryption.NewEncrypter(
    common.CipherSuite(...AESGCM), (*common.Key)(beginSegment.DerivedKey),
    &startingNonce, encryption.DefaultBlockSize)
paddedReader := encryption.PadReader(plainFile, encrypter.InBlockSize())
transformedReader := encryption.TransformReader(paddedReader, encrypter, 0)
```

This is the only place data is encrypted, and it runs **inside the uplink service** with the
coordinator-supplied `DerivedKey` ([`pkg/encryption`](../pkg/encryption)). Everything downstream
handles ciphertext only.

### 4.4 Decision A — inline vs. distributed

`if partSize <= fileInfo.InlineThreshold`:

- **Inline:** the uplink reads the whole ciphertext (`io.ReadAll(transformedReader)`) and sends it via
  `hubServer.File().MakeInlineSegment` (`EncryptedInlineData`). The coordinator's
  [`file.Service.MakeInlineSegment`](../coord/file/service.go) stores the encrypted bytes directly in
  the DB. No workers, no order limits consumed.

- **Distributed:** the uplink builds an `ers_schema.RedundancyStrategy` from
  `beginSegment.RedundancyScheme` and calls `PutSegment`.

### 4.5 Decision B — RS vs CLONE, and the split

[`UploadService.PutSegment`](../uplink/application/service/upload.go) pads the encrypted stream to the
stripe size and erasure-encodes it:

```go
pieceReaders, _ := eestream.EncodeReaderSimple(piecesCtx, paddedForErasure, rs)
```

using [`uplink/internal/eestream`](../uplink/internal/eestream) — one reader per piece. For
`REED_SOLOMON` these are erasure shares; for `CLONE` the pieces are full replicas and the uplink
additionally builds a Merkle tree over the encrypted bytes with [`merkle.Build`](../pkg/merkle) so the
root can be committed for later chunk audits.

### 4.6 Parallel piece upload — worker side

`PutSegment` launches one goroutine per `OrderLimit`, calling `workerClient.PutPiece(ol, reader,
privKey)`. On the worker,
[`PieceStoreEndpoint.Upload`](../worker/piecestore/endpoint.go) (a bidirectional gRPC stream):

1. **Authorizes before touching disk** —
   [`verifyOrderLimit(PUT)`](../worker/piecestore/verification.go): the limit must be action `PUT`,
   addressed to *this* worker (`WorkerId`), unexpired/in grace, carry a valid **coordinator
   signature** (resolved fail-closed through the [`worker/trust`](../worker/piecestore) pool via
   `trust.GetSignee` + `signatureCheck.VerifyOrderLimitSignature`), and present a **fresh serial**
   recorded in `usedSerials` (single-use replay protection).
2. Checks available space (`MaxConcurrentRequest`, `AvailableSpace`).
3. Opens `pieceBackend.Writer(...)` — the **`HashStoreBackend`** (log-structured
   [`worker/pkg/hashstore`](../worker/pkg/hashstore)).
4. Per streamed frame, `verifyOrder` checks the **uplink-signed** order's serial matches the limit,
   never decreases, and stays within the allowance.
5. On `Done`: `verifyPieceHash` against the writer's computed hash, `writer.Commit(PieceHeader)`, then
   the worker signs its **own** hash with
   [`signing.SignPieceHash(SignerFromFullIdentity(svc.identity), ...)`](../pkg/signing/sign.go) and
   returns it in `UploadResponse{ PieceId, PieceHash }`.

`PutSegment` collects results, cancels remaining puts once `OptimalThreshold` successes arrive, and
errors out if successes fall below `RequiredCount`.

### 4.7 Commit the segment — `CommitSegment`

The uplink calls `hubServer.File().CommitSegment` echoing the signed `SegmentId` blob and the
`UploadResult[]`. [`file.Endpoint.CommitSegment`](../coord/file/endpoint.go):

- reconstructs segment state from the signed `SegmentId` blob;
- for each piece, verifies the worker's signature with
  [`signing.VerifyPieceHashSignature`](../pkg/signing/verify.go) against the public key captured at the
  worker's check-in — **the worker signature is the proof the worker actually stored the piece**;
  pieces with a bad signature are dropped;
- requires at least `OptimalShares` valid pieces;
- for **CLONE**, stores `MerkleRoot` + `ChunkSize` — **unsigned** (see checkpoints below);
- calls [`file.Service.CommitSegment`](../coord/file/service.go) to persist the segment row.

### 4.8 Finish — `CommitFile`

After all parts, [`UploadService.CompletedMultipartUpload`](../uplink/application/service/upload.go)
validates the stored parts and calls `hubServer.File().CommitFile`.
[`file.Endpoint.CommitFile`](../coord/file/endpoint.go) is a cheap pending→ready flip that **trusts the
per-segment validation already done at `CommitSegment`** — it makes the file downloadable.

---

## 5. Verification & signature checkpoints

| # | Checkpoint | Who signs | Who verifies | Where |
| - | ---------- | --------- | ------------ | ----- |
| 1 | **Part checksum** | — | uplink (`stagePart` vs `partChecksum`) | `upload.go` |
| 2 | **PUT order limit** | coordinator (`SignOrderLimit`) | worker (`verifyOrderLimit`, via trust pool) | `verification.go` |
| 3 | **Order replay** | uplink (single-use order) | worker `usedSerials.Add` (serial single-use) | `verification.go` |
| 4 | **Per-frame order** | uplink (`PiecePrivateKey`) | worker `verifyOrder` (`VerifyUplinkOrderSignature`) | `verification.go` |
| 5 | **Piece hash (worker proof of storage)** | **worker** (`SignPieceHash`) | coordinator (`VerifyPieceHashSignature`) at `CommitSegment` | `endpoint.go` / `coord/file/endpoint.go` |
| 6 | **CLONE Merkle root** | *(uplink builds it; root committed)* | **not signature-verified at commit** | `coord/file/endpoint.go` |

**On checkpoint 6 — the CLONE `merkle_root` is stored UNSIGNED.** The coordinator stores only `root +
chunk size` (`MerkleCommitment{Root, ChunkSize}` in [`coord/file/service.go`](../coord/file/service.go))
and explicitly does **not** signature-verify it at `CommitSegment`. The reasoning, quoted from
[`coord/file/endpoint.go`](../coord/file/endpoint.go): *"the worker's bytes are already anchored by its
uplink-signed piece hash, so a chunk-proof mismatch on intact data is a client fault, not a worker
fault."* The trust anchor for CLONE replicas at write time is therefore checkpoint 5 (the worker's
signed piece hash), not the Merkle root. The root is used only later by the **audit** path
([`dataflow-audit.md`](dataflow-audit.md)), where a mismatch on intact bytes is classified as a
*client fault*.

---

## 6. Failure & edge cases

- **Checksum mismatch / wrong part size** — `stagePart` rejects before `BeginSegment`; nothing is sent.
- **Too few successful piece puts** — `PutSegment` errors if successes `< RequiredCount`; a warning is
  logged if `< RepairThreshold`. The segment is not committed.
- **Worker out of space / too many requests** — `Upload` returns `ResourceExhausted`; the uplink treats
  that piece as failed and relies on the remaining workers reaching `OptimalThreshold`.
- **Forged / replayed order limit** — rejected by `verifyOrderLimit`: bad coordinator signature →
  fail-closed via the trust pool; reused serial → `usedSerials.Add` error. A client cannot flush
  arbitrary bytes with its own key.
- **Duplicate part** — `UploadPart` rejects a part whose index already has a stored segment;
  `CommitSegment` / `MakeInlineSegment` are **idempotent** on the coordinator (retried commit for the
  same segment is a no-op).
- **Bad worker piece hash signature** — dropped at `CommitSegment`; if too many drop below
  `OptimalShares`, the commit fails.
- **Interrupted upload** — `RecoverInterruptedTasks` / `RecoverInterruptFiles` reconciles partial
  local state; the coordinator file stays PENDING until `CommitFile`.
- **Inline path** consumes no order limits and touches no worker; only `MakeInlineSegment` persists it.

---

## 7. Where to go next

- [`../coord/file`](../coord/file) — `CreateFile` / `BeginSegment` / `CommitSegment` / `CommitFile`,
  segments, redundancy scheme geometry, the `SegmentId` blob.
- [`../coord/orders`](../coord/orders) — how PUT order limits and the ephemeral piece key are minted
  and signed.
- [`../worker/piecestore`](../worker/piecestore) — the worker `Upload` endpoint, order-limit/order
  verification, and the `HashStoreBackend`.
- [`../pkg/eestream`](../pkg/eestream) — Reed-Solomon erasure encoding (`EncodeReaderSimple`).
- [`../pkg/merkle`](../pkg/merkle) — the Merkle tree used to commit CLONE replicas.
- [`../pkg/signing`](../pkg/signing) — `SignOrderLimit`, `SignPieceHash`, and the verification helpers.
- [`../pkg/encryption`](../pkg/encryption) — AES-GCM `NewEncrypter` / `PadReader` / `TransformReader`.
- [`dataflow-audit.md`](dataflow-audit.md) — the proof-of-storage audit path (consumes what upload
  wrote).
- [`../SOURCE_STRUCTURE.md`](../SOURCE_STRUCTURE.md) — the layered-peer architecture every subsystem
  follows.
