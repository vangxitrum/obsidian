---
type: fact
tags: [ffmpeg, transcode, worker, bug-fix, hdr, pix-fmt, x264]
created: 2026-07-14
agent: main
---

Fixed a job-worker crash: 10-bit sources with a plain SDR transfer curve (bt709, not
PQ/HLG) failed to transcode with `x264 [error]: main profile doesn't support a bit
depth of 10` / exit 234. Real production case: an anime mkv (HEVC `yuv420p10le`,
`color_trc=bt709`) being transcoded to H.264 for HLS.

**Why the existing HDR fix didn't catch it:** `internal/core/worker/probe.go` already
had `probeIsHDR` + `hdrToneMapFilter` + `buildVideoFilterChain` (untracked/new as of
[[gosdk-system-headers-upload]]'s session, itself a fix for the *HDR* case of this
exact same class of bug - see `probe_test.go`'s `TestHDREncode_OldFilterCrashes_
NewFilterSucceeds`). But `probeIsHDR` only checks `ColorTransfer` against
`{smpte2084, arib-std-b67}` (PQ/HLG). A 10-bit source with `bt709` transfer is not
HDR by that check, so `isHDR=false`, the tone-map/`format=yuv420p` filter never runs,
and `-profile:v main` (hardcoded unconditionally in `worker.go:423`, applied to BOTH
`libx264` and `libx265` output regardless of codec) then rejects the 10-bit stream.
`models.Stream.PixFmt` was already being ffprobed and available - just never checked
independent of HDR-ness.

**Fix:** added `probeIsHighBitDepth` (probe.go) - same ffprobe pattern as `probeIsHDR`,
checks `PixFmt` against `highBitDepthPixFmtSuffixes` (`p9/10/12/14/16le|be`). Extracted
the shared ffprobe-and-decode boilerplate into `probeFirstVideoStream` so both probes
reuse it without duplicating the shell-out. `buildVideoFilterChain` gained an
`isHighBitDepth bool` param: appends a bare `format=yuv420p` after the scale filter
when the source is high-bit-depth but NOT HDR (HDR's tone-map chain already ends in
`format=yuv420p`, so no double-append). `worker.go`'s `generateCmd` probes both booleans
once per job and also folds `isHighBitDepth` into the GPU-fallback gate (`canRunWithGpu
= canRunWithGpu && !isHDR && !isHighBitDepth`) - the CUDA/nvenc path has no pix-fmt-
downconvert equivalent either, same reasoning as the existing HDR fallback.

**Verified end-to-end before fixing** (per CLAUDE.md's bug-fix rule): a fork built a
synthetic 10-bit `yuv420p10le`/`bt709` fixture
with real `ffmpeg`, reproduced the exact `x264 main profile` error against the real
failing command shape, then confirmed `format=yuv420p` in the filter chain fixes it and
is a no-op passthrough on already-8-bit sources. Added permanent regression coverage
mirroring the existing HDR test exactly: `generateHighBitDepthSDRFixture`,
`TestProbeIsHighBitDepth`, `TestHighBitDepthSDREncode_OldFilterCrashes_
NewFilterSucceeds` in `probe_test.go` - all shell out to real `ffmpeg`/`ffprobe`
(skipped via `requireBinary` if unavailable). All probe.go/worker.go tests green
post-fix; `go build`/`go vet` clean.

**Takeaway:** when adding source-format-conditional filter logic (HDR, bit depth, etc.)
to this transcode pipeline, check ALL the ways a "main profile only supports 8-bit"
violation can occur, not just the one that prompted the fix - HDR and high-bit-depth
are correlated but independent axes (this source was 10-bit SDR, i.e. high-bit-depth
without HDR).
