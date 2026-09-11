---
type: fact
tags: [go-livepeer-openpool, livepeer, lpms, codecs, verification, ffmpeg]
created: 2026-08-25
agent: main
---

Codec support + output-verification map for go-livepeer (upstream v0.8.10; the
openpool fork changes NEITHER - `git diff master origin/release/open-pool/v0.8.10
-- verification/` is empty).

LPMS pinned at `github.com/livepeer/lpms v0.0.0-20260310011836-c44af7253bc9`
(go.mod:22). **It is not extracted in the local module cache** (no zip under
`~/go/pkg/mod/cache/download/github.com/livepeer/lpms/@v/`, no `vendor/`), so LPMS
source must be read from `raw.githubusercontent.com/livepeer/lpms/c44af7253bc9/...`.

## Codecs

`VideoCodec` enum is closed at four: H264=0, H265=1, VP8=2, VP9=3
(`lpms:ffmpeg/videoprofile.go:55-62`). **No AV1 anywhere in either repo.**

Encoder matrix, `lpms:ffmpeg/ffmpeg.go:66-81` `FfEncoderLookup`:
- Software: libx264, libx265, libvpx, libvpx-vp9
- **Nvidia: h264_nvenc, hevc_nvenc ONLY**
- **Netint: h264_ni_enc, h265_ni_enc ONLY**
- `Amd` accel is declared in the enum but has no encoder table and no
  `accelDeviceType` case - non-functional.
- No QSV / VAAPI / AMF at all.

Non-obvious gotchas:
- VP8/VP9 on Nvidia/Netint yields an **empty encoder string, not an error**.
- `nvidiaCodecSizeLimts` only covers H264/H265 but the `clamp()` runs for every
  accel, so VP8/VP9 clamp against a zero-valued limits struct (`ffmpeg.go:596-600`).
- `CodecNameToValue` (`videoprofile.go:210-220`) is **exact and case-sensitive**:
  only `"H.264"`, `"HEVC"`, `"VP8"`, `"VP9"` parse. `"h264"`/`"hevc"` -> ErrCodecName.
  (Input-side `FfmpegNameToVideoCodec` uses the lowercase forms - different map.)
- `Capability_VP8_Encode`/`VP9_Encode` are in neither `DefaultCapabilities()` nor
  `OptionalCapabilities()` (core/capabilities.go:187-230), and have no
  `CapabilityTestLookup` probe - and untested caps are **assumed supported**
  (core/transcoder.go:287-291).
- `server/segment_rpc.go:335-397 makeFfmpegVideoProfiles` drops `ColorDepth` and
  `ChromaFormat` from the wire even though `common/util.go:180-181` sends them.
  10-bit / 4:2:2 requests silently degrade on the orchestrator.

Containers: mpegts + mp4 only. Profiles: H.264 baseline/main/high/constrainedhigh
only. Audio: no enum; orchestrator always transcodes with `"copy"`
(core/transcoder.go:454); lpms defaults to `"aac"`; `"drop"` is a sentinel.

## Output verification - who checks what

**Orchestrator, output side (thin):** segment count == profile count
(core/orchestrator.go:755), `len(Data) >= 25` (:773), huge-output heuristic logs
only (:790), missing PHash is **logged not enforced** - the `return terr()` is
commented out with a FIXME (:777-781). Signs Keccak256 of the rendition hashes,
**skipped entirely when `n.Eth == nil`** (:802-812).

**Orchestrator NEVER re-derives pixel counts.** `server/ot_rpc.go:409-417` trusts
the remote transcoder's `Pixels` multipart/HTTP header verbatim, and
`segment_rpc.go:206` bills `DebitFees` from it. `countPixels` exists only at
`verification/verify.go:235`, which is gateway-side.

**Gateway (`verification.SegmentVerifier`), default ON onchain** - `-localVerify`
defaults true (starter.go:290), forced false offchain unless the flag is explicit
(:1684-1689). Does sig check + **full local re-decode pixel count** per rendition
per segment, Retries:2. Gotcha: `sigVerification` silently returns nil if
Orchestrator/Address/TicketParams is nil (verify.go:187-189). Second gotcha:
rendition bytes are only downloaded at all when `verifier != nil`
(broadcast.go:1332-1348).

**Opt-in `-verifierUrl`:** Epic classifier (verification/epic.go) - tamper score,
audio distance (Fatal), OCSVM score.

**Opt-in "fast verification":** 1 trusted + 2 untrusted orchestrators, MPEG-7
signature compare then full video compare on ONE random rendition
(broadcast.go:613-789). **Enabled only via the auth webhook's `verificationFreq`
field - there is no CLI flag for it.**

**Declared but dead:** `Policy.SampleRate` and `Policy.Redundancy`
(verification/verify.go:87-90) are never read; the verifier runs on every segment.

**Never checked anywhere:** that a rendition's actual resolution / framerate /
codec / container matches the requested VideoProfile.

## Why this matters for the pool

See [[pool-attribution-architecture]]. `OnTranscode` computes `computeUnits` and
`fees` from exactly the worker-self-reported `Pixels` header that nobody on the
orchestrator verifies. The only party that re-decodes is the gateway, whose remedy
is `removeSession` (drop the whole orchestrator) - it has no path back into the
pool event stream, and the orchestrator cannot attribute a gateway-rejected segment
to the specific worker that produced it.
