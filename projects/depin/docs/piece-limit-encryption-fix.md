# `pieceLimit` undercounts encrypted shard size

## Symptom

Worker rejects an upload because the coordinator-authorized order limit is
smaller than the shard the uplink actually produces:

```
size:  2343168   // real per-piece bytes the uplink uploads
limit: 2314496   // per-piece authorization stamped into the OrderLimit
                 // 2343168 > 2314496  -> worker rejects
```

## Root cause

`pieceLimit` (coordinator, `coord/application/usecase/file.go`) computed the
authorization from the **plaintext** `SegmentSize`. But the uplink **encrypts
before erasure coding**, and AES-GCM expands the data. The limit never accounted
for that expansion.

### The size pipeline (uplink-sdk)

`uplink-sdk/internal/segmentupload/segment.go` does, in order:

1. **Encryption pad** — `PadReader(plaintext, InBlockSize)`
2. **Encrypt** — `TransformReader(..., encrypter)` → every `InBlockSize` block becomes `BlockSize` bytes
3. **Erasure pad** — `PadReader(encrypted, stripeSize)`
4. **Erasure encode** — each piece = `numStripes × shareSize`

The key detail: in `NewAESGCMEncrypter`, the configured `BlockSize` is the
**encrypted (output)** block size, so the _plaintext_ block is smaller:

```
InBlockSize  = BlockSize - 16   (16 = AES-GCM auth tag, per block)
OutBlockSize = BlockSize
```

With the coordinator's `BlockSize = 256`:

```
InBlockSize = 240,  OutBlockSize = 256   ->  256/240 = 6.7% expansion
```

That 6.7% is **per block**, not a one-time overhead — for a 64 MiB segment it is
~600 stripes, not "less than one stripe."

### Worked numbers (`SegmentSize = 64 MiB`, `ShareSize = 256`, `k = 29`)

```
stripeSize = k × shareSize = 29 × 256 = 7424

Encrypted size:
  InBlockSize = 256 - 16 = 240
  blocks      = ceil((67,108,864 + 4) / 240) = 279,621
  encrypted   = 279,621 × 256                = 71,582,976 bytes

Per-piece stripes:
  numStripes  = ceil((71,582,976 + 4) / 7424) = 9643
```

| Quantity                       | Old (plaintext-based) | New (encrypted-based) |
| ------------------------------ | --------------------: | --------------------: |
| stripes / piece                |                  9041 |              **9643** |
| limit / piece                  |             2,314,496 |         **2,468,608** |
| covers real shard `2,343,168`? |                 ❌ no |                ✅ yes |

## Fix

Compute the authorization from the **encrypted** segment size, mirroring the
uplink's `encryption.CalcEncryptedSize`, then erasure-pad.

`coord/application/usecase/file.go` — before stamping order limits:

```go
encParams := entity.EncryptionParameters{
    CipherSuite: sharedpb.CipherSuite_ENC_AESGCM,
    BlockSize:   256,
}

// Worst-case encrypted size for a full segment.
encryptedSegmentSize, err := encryption.CalcEncryptedSize(
    storageCfg.SegmentSize,
    common.EncryptionParameters(encParams),
)
if err != nil {
    return nil, SegmentError.Wrap(err)
}
```

`pieceLimit` now takes the encrypted size and includes the erasure-pad trailer:

```go
const erasurePadTrailer = 4 // 4-byte length trailer added by PadReader

func pieceLimit(encryptedSize, shareSize, requiredShares int64) int64 {
    if requiredShares <= 0 || shareSize <= 0 {
        return shareSize
    }
    stripeSize := requiredShares * shareSize
    numStripes := (encryptedSize + erasurePadTrailer + stripeSize - 1) / stripeSize
    return max(numStripes, 1) * shareSize
}
```

The same `encParams` is reused for `NewSegment`, so the value the limit is
computed from is guaranteed to match what the uplink actually applies.

Because the limit is derived from the **maximum** `SegmentSize`, it is an upper
bound that covers any smaller real segment.

## Why the earlier "+1 stripe" attempt was wrong

The first patch added one stripe of headroom, assuming encryption padding is
always `< 1` stripe. That only holds when the cipher block is large relative to
the stripe. With `BlockSize = 256` the expansion is **proportional** (6.7% of the
whole segment), so the shortfall scales with segment size — one stripe could
never cover it.

## Tests

`coord/application/usecase/file_test.go`:

- `TestPieceLimit` — stripe-rounding incl. the +4 pad trailer and fallbacks.
- `TestPieceLimitCoversEncryptedSegment` — regression guard: the limit computed
  from the max `SegmentSize` is `>=` the real per-piece size for plaintext sizes
  from 1 byte up to the cap.
