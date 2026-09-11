---
title: Wide-input-range VOD support for the aioz-stream-core worker
date: 2026-08-25
project: aioz-stream
tags: [plan, vod, ffmpeg, transcoding, hls, hevc]
---

# Wide-input-range VOD support for the aioz-stream-core worker

## Context

`internal/core/worker/worker.go` builds ffmpeg CLI args from a `models.Job` that the
gRPC caller supplies. Today it works for the happy path: a single-stream, 8-bit,
4:2:0, CFR, unrotated, square-pixel h264/h265 MP4 with audio at stream 0. Anything
outside that either fails with a generic `failed to execute CPU command` or silently
produces wrong output.

Confirmed by exploration:

- **No probe exists anywhere in the repo.** `grep -riE "ffprobe|mediainfo|probe"` finds
  nothing. `internal/models/stream.go:9-83` is a complete `ffprobe -show_streams` JSON
  mirror that nothing ever fills. The upstream API/upload service does the analysis.
- The worker downloads bytes to `<storage>/<jobId>/input/source`, MD5-checks the
  *transfer*, and immediately generates args. It never verifies decodability.
- `models.PlaylistConfig.IsValid()` is `return true` with a TODO
  (`internal/models/input.go:18-21`), so nothing is rejected server-side either.

**Decisions taken (from the user):**

1. Probing stays **upstream**; the worker validates the caller-supplied config against
   the real file and fails fast on mismatch. Ladder/rendition planning stays upstream.
2. Scope is **input robustness only** - no subtitles, thumbnails, watermark, or
   per-title encoding in this plan.
3. Stay on the **ffmpeg CLI** (`exec.Command`). No cgo/libav port. All fixes are
   expressed as arg generation + fallback logic.

Reference implementation studied: LPMS (`/home/tuan/work/depin-workspace/lpms`), which
solves the same problems in-process via libav. Its *decisions* port to the CLI even
though its mechanism does not.

**Goal:** an arbitrary user upload - 10-bit HDR HEVC, a rotated iPhone MOV, a 5.1 MKV
with video on stream 3, a VFR screen recording, a 4:3 anamorphic DVD rip, an
audio-only file, a 240fps GoPro clip - transcodes to a correct HLS/DASH ladder or
fails with a precise, classified error.

---

## What to add, in priority order

### P0 - Correctness bugs that break whole input families

**1. Stream mapping is wrong for any input whose video is not stream 0**

`worker.go:697` (`-map [svid]` via `[%d:v]` at `worker.go:676`/`689`) and
`worker.go:754` (`-map %d:v:0`) interpolate `VideoConfig.Index` as ffmpeg's
**input-file** index. Audio at `worker.go:413` uses `0:a:%d`, where it is the
**stream** index. With one `-i`, any `VideoConfig.Index != 0` produces
`Invalid file index` / `matches no streams`. Multi-track MKV and MOV routinely put
video at index 1+.

Fix: `-map 0:v:%d` and `[0:v:%d]` in the filter_complex label; keep audio as-is.

**2. Non-video streams are never excluded**

Add `-sn -dn -map_chapters -1 -ignore_unknown` (and `-map_metadata -1` unless
metadata is wanted). MKV font attachments and data streams currently reach the
HLS/TS muxer and abort it.

**3. No `-pix_fmt` is ever set**

A 10-bit or 4:2:2 source down the CPU path hits `libx264` while `-profile:v main` is
asserted unconditionally at `worker.go:360` - hard failure. Set `-pix_fmt yuv420p`
for 8-bit output, `p010le`/`yuv420p10le` only when the rung explicitly wants Main10.
Make `-profile:v` codec-aware and take it from the ladder rung's `Profile` field
(`internal/models/quality_config.go:117-128` already carries `Profile`/`ProfileId`
per rung and nothing reads them).

**4. `GenerateSegmentConfig` emits invalid muxer options** (`internal/models/job.go:145-183`)

- `-hls_playlist true` is a **DASH-muxer** option; it is not valid for `-f hls`.
- `-hls_segment_type` accepts only `mpegts|fmp4`; a caller sending
  `ContainerType:"mp4"` yields `-hls_segment_type mp4`.
- `ContainerType:"fmp4"` maps to `ext=""` (job.go:147-152), so segments are written
  as `segment_000` with no extension.
- `-dash_segment_type` accepts only `auto|mp4|webm`; `"mpegts"`/`"fmp4"` are invalid.
- Any `SegmentType` outside `hls|dash|mp4` returns `nil` args, so ffmpeg runs with
  **no output file**.
- `-hls_segment_filename` has no `%v` while `-var_stream_map` is passed - latent
  failure the moment a playlist carries more than one variant.
- MP4 branch has no `-movflags +faststart`.

**5. `GetVideoResolutionConfig` divides by zero and mixes axes**
(`internal/models/quality_config.go:184-219`)

- `vWidth==0 || vHeight==0` (missing caller metadata) falls into the else branch and
  divides by `vHeight` -> `NaN` -> garbage `scale=` value.
- The portrait branch at `:210-214` uses `width = c.Height` and compares
  `height != c.Width`, mixing ladder width and height.

**6. `-initial_offset` is not a global ffmpeg option**

`worker.go:376-382` / `404-410` emit it as a global flag; it is an HLS-muxer option
and must follow `-f hls`. Worse, `cpuArgs` is snapshotted at `worker.go:361`
*before* both blocks, so the video offset reaches only the GPU command and the audio
offset reaches **neither**. Decide the intent (trim vs. playlist offset) and use
`-ss` or `-hls_start_number` accordingly.

**7. `AudioConfig.SilenceDuration` is dead code**

The `-af adelay=` branch at `worker.go:419-428` can never fire: `SilenceDuration`
exists in `internal/models/job.go:185-197` but not in `internal/proto/job.proto:232-243`.
Add it to the proto or delete the branch.

**8. Nil deref in dry run** - `worker.go:311` calls `pl.GpuCmd.String()` with no nil
check; panics whenever no GPU command was generated.

---

### P1 - Input-verification layer in the worker

The worker still needs to *see* the file to validate against it. One `ffprobe` call
per job, after download, before `generateCmd`:

```
ffprobe -v error -show_streams -show_format -show_entries stream_side_data \
        -print_format json <storage>/<jobId>/input/source
```

**Reuse `internal/models/stream.go:9-83`** - it is already the exact ffprobe schema.
Do not write new structs.

Then:

**9. Validate the caller's config against reality.** Reject before spending GPU time:
declared stream indices exist and have the declared `codec_type`; declared
`Width`/`Height` match `codec_width`/`codec_height`; declared codec matches;
`format.duration > 0`; crop rect fits inside the source (`VideoCropInfo.Validate` at
`internal/models/job.go:120-133` only checks for negatives - it never compares against
source dimensions).

Use the already-declared-but-unused `models.UnsupportedCodecErr` / `models.NilVideoErr`
(`internal/models/variable.go:130-131`).

**10. Classify degenerate inputs**, mirroring LPMS `lpms_get_codec_info`
(`ffmpeg/extras.c:147-201`):

| State | LPMS detection | Action |
|---|---|---|
| streams missing | neither `av_find_best_stream` succeeds (extras.c:163-166) | fail fast, non-retryable |
| needs bypass | video stream present but `pix_fmt == NONE && height == 0` (extras.c:177-182) | treat as audio-only |
| no keyframes | `lpms_ERR_INPUT_NOKF` (`ffmpeg/decoder.c:60`, surfaced `transcoder.c:364-365`) | fail fast, non-retryable |
| ok | otherwise | transcode |

LPMS's own reasoning for the bypass state is in `doc/quirks.md:4-18`. The
no-keyframe case has a CLI equivalent: `ffprobe -select_streams v -show_frames
-show_entries frame=key_frame -read_intervals "%+#1"` on the first packets, or
simply `nb_read_frames` with `-skip_frame nokey`.

**Note on duration:** LPMS caps input at 300 s (`ffmpeg/ffmpeg.go:884-887`,
`ErrTranscoderDuration`). That is a live-segment constraint - do **not** port it to
VOD. Still add *some* upper bound so a malformed duration cannot produce an
unbounded job.

**11. Add the fields the worker cannot guess to the proto**, so upstream fills them
and the worker only checks them (per decision #1). At minimum on `VideoConfig`:
`pix_fmt`, `color_primaries`, `color_transfer`, `color_space`, `color_range`,
`rotation` (from `side_data_list`), `sample_aspect_ratio`, `nb_frames`,
`r_frame_rate` **and** `avg_frame_rate` (their divergence is the VFR signal),
`has_b_frames`, `codec_tag`. Note `VideoConfig.Sar` already exists in
`internal/models/job.go:101-118` and is never used.

---

### P2 - Pixel format, bit depth, colour

**12. GPU gating is broken for 4:2:2 / 4:4:4 and blind to 10-bit**

`EncoderConfig.IsSupported` (`internal/models/worker.go:82-111`) matches the literals
`"yuv422"` / `"yuv444"`, but the constants are `models.Yuv422 = "yuv422p"` and
`Yuv444 = "yuv444p"` (`internal/models/variable.go:98-99`) - they can never match.
The struct also has `H265_10bit`, `H265_8K`, `H265BFrame` fields (`worker.go:77-79`)
that `IsSupported` never consults. And only the exact literals `"h264"`/`"h265"` are
accepted, so `"hevc"`, `"H264"`, `"av1"`, `""` all silently fall to CPU.

Fix the constants, consult the bit-depth fields, and normalise codec names before
lookup.

**13. Route non-4:2:0 away from NVDEC.** NVDEC cannot decode 4:2:2/4:4:4 or 12-bit at
all. LPMS hard-rejects this on the hw path - `decoder.c:279-286` returns
`lpms_ERR_INPUT_PIXFMT` ("Non 4:2:0 pixel format detected in input"). Mirror the rule:
non-4:2:0 source -> CPU decode.

**14. HDR handling.** Detect `bt2020nc` / `smpte2084` / `arib-std-b67`. Currently no
colour metadata is passed at all, so HDR sources come out washed out. Pick one and
implement it:

- *Preserve:* `-color_primaries bt2020 -color_trc smpte2084 -colorspace bt2020nc`
  plus `-x265-params hdr-opt=1:repeat-headers=1:master-display=...:max-cll=...`.
- *Tonemap to SDR:* `zscale=t=linear:npl=100,tonemap=hable,zscale=p=bt709:t=bt709:m=bt709:r=tv,format=yuv420p`.

Recommendation: tonemap to SDR for the standard ladder, preserve only on rungs that
explicitly declare a 10-bit HDR profile. LPMS carries colourspace and range into the
filtergraph explicitly (`ffmpeg/filter.c:82-83`) but never touches
`color_primaries`/`color_trc` at all - so HDR10 input there is silently squashed to
8-bit BT.709-tagged output. That is the failure mode to avoid, not to copy.

**14b. RGB / GBRP / paletted sources.** LPMS's pixel-format classifier rejects
everything outside YUV 420/422/444 at 8/10/12/16-bit with
`ErrTranscoderPixelformat` (`ffmpeg/ffmpeg.go:181-236`, default case). Screen
recordings, ProRes 4444, PNG/image sequences and some AVI carry `rgb24`/`gbrp`/`pal8`.
Handle them by forcing `format=yuv420p` at the head of the filter chain rather than
rejecting - LPMS itself does this for PNG on Nvidia (`ffmpeg/ffmpeg.go:644-648`,
`format=nv12,` prefix, because `scale_npp` cannot ingest RGB). The same applies to
`scale_cuda`.

---

### P3 - Geometry: rotation, SAR, encoder size limits

**15. Rotation.** ffmpeg's autorotate does not survive `-filter_complex` with
`-hwaccel cuda`, and `scale_cuda` bypasses it entirely. Read
`side_data_list[].rotation`, insert `transpose=1|2` / `hflip,vflip`, and **swap the
target W/H** when rotation is ±90. Without this every portrait phone upload is
letterboxed sideways.

**16. Anamorphic SAR.** `setsar=1` at `worker.go:689` discards the sample aspect
ratio, so 1440x1080 HDV and DVD rips come out horizontally squashed. Compute the
display resolution from SAR *before* choosing the ladder rung, then scale to square
pixels.

**17. Even dimensions and encoder size limits.** The `-2` trick in
`GetVideoResolutionConfig` is right in spirit; LPMS is stricter and worth copying:

- Scale expressions that clamp *and* force even:
  `trunc(...)/2)*2` (`ffmpeg/ffmpeg.go:623-635`).
- Hard encoder limits, applied as a clamp before building the expression
  (`ffmpeg/ffmpeg.go:555-558`, `599-601`):
  `H264 {WidthMin:146, HeightMin:50, WidthMax:4096, HeightMax:4096}`,
  `H265 {132, 40, 8192, 8192}`. NVENC fails outright outside these.

---

### P4 - Frame rate and timestamps

**18. No frame-rate control exists at all** - no `-r`, no `fps` filter, no
`-vsync`/`-fps_mode`. VFR sources (screen recordings, WebM, GoPro) drift and desync.
Add `-fps_mode cfr` plus an `fps=` filter, and place the `fps` filter **after**
`scale` - LPMS explains why at `ffmpeg/ffmpeg.go:653-661`: putting it first means
scaling duplicated frames, which is a DoS vector when two frames are far apart in PTS.

**19. Cap the frame rate.** A 240fps GoPro clip currently encodes every rung at
240fps. Add `fps=min(source_fps,60)`.

**19b. Guard against fps-filter frame explosion.** Two frames far apart in PTS make
the `fps` filter duplicate frames to fill the gap - a DoS vector, and the reason LPMS
puts `fps` after `scale`. LPMS adds a hard abort: if encoded frames exceed 25x decoded
frames it fails with `lpms_ERR_ENC_RUNAWAY` / "Encoded frames runaway"
(`ffmpeg/encoder.c:667-682`, exempting image2 input). CLI equivalent: bound the job
with `-frames:v <ceil(duration * target_fps * 1.1)>` and treat a rendition whose
duration far exceeds the source as a failure.

**20. Demuxer hardening.** `-fflags +genpts`, `-avoid_negative_ts make_zero`,
`-max_muxing_queue_size 4096`, and larger `-probesize` / `-analyzeduration` for
long-GOP MPEG-TS and extension-less input (the source is saved as `input/source`
with no extension at `worker.go:180`). Where the probe identifies the container,
pass `-f <fmt>` explicitly.

---

### P5 - GOP and segment alignment

**21. No `-g`, `-keyint_min`, `-sc_threshold`, or `-force_key_frames` anywhere.**

Consequences: actual segment durations drift from `SegmentConfig.Duration`, and GPU
and CPU renditions of the same ladder are not keyframe-aligned, so ABR switching
breaks. The repo's own reference commands do it right (`cmd1`: `-keyint_min 100 -g 100`);
the Go code does not.

Prefer the fps-independent form, since fps may be unknown or variable:

```
-force_key_frames "expr:gte(t,n_forced*<SegmentConfig.Duration>)"
```

plus `-sc_threshold 0` on x264/x265 and `-forced-idr 1 -no-scenecut 1` on NVENC.
LPMS sets `forced-idr=1` by default and derives `g` from GOP x framerate
(`ffmpeg/ffmpeg.go:674-678`, `730-747`).

---

### P6 - Audio robustness

**22. Normalise the audio target instead of trusting the caller.** LPMS forces a
single audio format for every output: `aformat=sample_fmts=fltp:channel_layouts=stereo:sample_rates=44100`
(`ffmpeg/filter.c:175`). The worker passes `-ac`/`-ar` straight through from
`AudioConfig` with no validation (`worker.go:416-417`), so a 5.1 source with `-ac 2`
and no downmix filter, or an 8kHz source, reaches the muxer as-is.

Add: `-af aformat=channel_layouts=stereo,aresample=async=1:first_pts=0`, clamp
`-ar` to 44100/48000, and whitelist `-c:a` (aac/opus) instead of interpolating
whatever string arrived (`worker.go:414`).

**23. Declared-but-absent audio.** `-map 0:a:%d` on a source with no audio is a hard
failure. Use the optional specifier `0:a:%d?`, and when a rung requires audio the
source lacks, synthesise it:
`-f lavfi -i anullsrc=channel_layout=stereo:sample_rate=48000 -shortest`.

**24. Audio-only playlists still get video args.** When `VideoConfig == nil` the
worker still appends `-preset fast -profile:v main` (`worker.go:359-360`) and the HLS
block still passes `-master_pl_name`. Guard both.

---

### P7 - Decoder coverage and GPU fallback

**25. Support a third acceleration mode: software decode + GPU encode.**

The GPU command is currently all-or-nothing - `-hwaccel cuda -hwaccel_output_format cuda`
prepended at `worker.go:386-389`. NVDEC cannot decode AV1 on pre-Ampere, VP9 profile 2,
ProRes, DNxHD, or non-4:2:0 of anything. Those inputs today either fail on GPU or take
the fully-software path and lose the encoder.

LPMS models exactly this matrix in `configEncoder` (`ffmpeg/ffmpeg.go:478-515`); for
software-in/Nvidia-out it emits `hwupload_cuda` + hw scale (`:487-493`), and for
Nvidia-in/software-out it appends `hwdownload,format=nv12` (`:640-643`).
Port the matrix: `{sw,nv} x {sw,nv}`, four arg templates instead of two.

**26. Verify decoder and demuxer availability at build time.** This is the single
biggest determinant of input range and it is a build gap, not a code gap. LPMS makes
the point sharply: `install_ffmpeg.sh:212-236` starts from `--disable-demuxers
--disable-decoders ...` and re-enables a narrow allowlist - production LPMS has **no
avi, no ogg, no mpeg-ps demuxer**, and **no libx265 or libvpx encoder** at all.

The aioz `models.MimetypeMapping` (`internal/models/variable.go:50-63`) advertises
mp4/mkv/avi/mov/flv/webm/m4v/**wmv**/**asf**/**mpg**/f4v/**ogv**. Confirm the worker's
ffmpeg actually carries those demuxers plus decoders for mpeg2video, mpeg4/xvid,
vc1/wmv3, prores, dnxhd, theora, mjpeg, av1 (libdav1d), vp8/vp9. Add a startup
assertion that fails loudly rather than accepting jobs it cannot decode.

**26b. Misplaced SEI in MPEG-TS h264.** LPMS carries a pure-Go fixer:
`ffmpeg/sei_fixup.go` parses PAT/PMT to find the H.264 PID, reassembles the elementary
stream, and reorders SEI (type 6) NALs that appear *after* the first VCL NAL back in
front of it, splitting access units on AUD. Invoked only for `mpegts` + `h264`
(`ffmpeg/ffmpeg.go:915-922`). If TS uploads are in scope, this file is portable
as-is - it is standalone Go over `livepeer/joy4`, with no cgo.

**26c. Mid-stream resolution or orientation change.** Concatenated or broadcast-sourced
files change dimensions mid-stream. LPMS handles it by re-initialising the encoder
(`ffmpeg/encoder.c:350-419`) and filtergraph (`ffmpeg/filter.c:291-332`). The ffmpeg
CLI cannot do this - it will fail or produce garbage. Detect it at probe time
(differing `codec_width` vs `width`, or multiple video streams) and either reject or
pre-normalise with a separate remux pass. This is the one item on the list the CLI
genuinely cannot cover.

**27. HEVC-on-iOS: replace Bento4 with ffmpeg + a correct master playlist.**

Bento4 was introduced to stop h265 HLS showing a black screen on iOS. It does not
currently fix that, because the breakage is in the Go playlist generator that runs
*after* Bento4, and Bento4's own output is discarded.

Apple's HEVC-in-HLS requirements, and where each stands today:

| Requirement | Status |
|---|---|
| HEVC in **fMP4 only** (never MPEG-TS) | caller-controlled via `ContainerType`; nothing enforces it |
| `hvc1` sample entry, not `hev1` | **OK** - `-tag:v hvc1` at `worker.go:656` (GPU) and `:731` (CPU) |
| valid RFC6381 `CODECS` in the master playlist | **broken** (below) |
| `EXT-X-VERSION` >= 7 for `EXT-X-MAP` | **broken** - `m3u8_helper/helper.go:32` hardcodes `SetVersion(3)` |

The CODECS breakage: `m3u8_helper/helper.go:45` sets `variant.Codecs = pl.VideoCodec`,
but `pl.VideoCodec` is assigned only inside the **DASH** parse branch
(`path_manager.go:378-381`). HLS jobs leave it as the raw DB `v_codec` value - the
literal string `"h265"`. Safari rejects a variant whose CODECS is not a valid RFC6381
string. And when `GetCodecName` does run (`internal/utils/codec/utils.go:10-35`) it
produces `hev1.1.6.L153.B0` - the wrong prefix versus the `hvc1` tag ffmpeg wrote, and
derived from the `models.DefaultProfile` / `DefaultH265Level` globals rather than the
rung's actual `-level:v`.

Two further inconsistencies in the same file: `helper.go:56-59` sets
`variant.VariantParams.Audio = "audio"` (lowercase) only when `AudioConfig == nil`,
while the alternatives use `GroupId: "AUDIO"` (uppercase) - group ids never match; and
only `firstVariant` receives the alternatives list (`helper.go:80-82`).

Meanwhile Bento4 itself is doubly ineffective:

- **Its master playlist is discarded.** `MergeM3U8Files` writes its own `master.m3u8`
  at the job root, so `mp4hls`'s correctly-derived CODECS never reaches the player.
- **It may not be installed.** `./bento4/bin/mp4fragment` is a relative path and
  `Dockerfile.worker` does not install Bento4 - the base image is
  `tuanaioz/ffmpeg:latest`; confirm whether it carries the binaries.
- **The GPU path bypasses it entirely.** `RunWithBento4()`
  (`internal/models/quality.go:121-124`) gates only the CPU branch;
  `worker.go:495-500` appends raw ffmpeg HLS args regardless. So h265+HLS yields two
  different output layouts depending on which device ran the job, while
  `path_manager.go:147` expects `playlist.m3u8` in the playlist dir either way. If GPU
  h265 plays correctly on iOS today, that is direct evidence ffmpeg alone suffices.

**Recommendation: drop Bento4.** Use ffmpeg's fMP4 HLS muxer -

```
-c:v libx265 -tag:v hvc1 -pix_fmt yuv420p \
-f hls -hls_segment_type fmp4 -hls_fmp4_init_filename init.mp4 \
-hls_playlist_type vod -hls_time <SegmentConfig.Duration>
```

- and fix the Go generator: `SetVersion(7)`, and build CODECS from the encode that
actually happened rather than from globals. The one thing Bento4 does that ffmpeg does
not is derive the codec string from the real `hvcC` box; the cheap equivalent is to
`ffprobe -show_entries stream=codec_tag_string,profile,level` the generated `init.mp4`
and construct `hvc1.<profile>.<flags>.L<level*30>.B0` from that. Note LPMS offers no
prior art here - it never sets a codec tag, never builds CMAF, and never emits a CODECS
attribute (`ffmpeg/encoder.c:25`, `ffmpeg/ffmpeg.go:760-764`,
`ffmpeg/videoprofile.go:175-186`).

Validate with Apple's `mediastreamvalidator` against the merged master playlist, not
against the per-playlist output.

**28. Make the GPU->CPU fallback diagnose instead of swallow.**

Worth noting what LPMS does *not* do here: it has **no software fallback at all**. A
wedged CUDA context (`AVERROR_UNKNOWN`) becomes `lpms_ERR_UNRECOVERABLE` and the Go
layer **panics** (`ffmpeg/decoder.c:321-324` -> `ffmpeg/ffmpeg.go:1031-1033`) so the
supervisor restarts the process. Its contribution is not a fallback mechanism but a
**classification contract** for the caller, and that is the part worth porting.

`Playlist.Exec` (`internal/models/quality.go:55-119`) already falls back on any GPU
error, but logs at **Debug**, never inspects the exit code, and never parses stderr.
Add error classification. Port LPMS's non-retryable list verbatim
(`ffmpeg/ffmpeg_errors.go:54-78`):

> `"Decoder not found"`, `"Demuxer not found"`, `"Encoder not found"`,
> `"Muxer not found"`, `"Option not found"`, `"Invalid argument"`, plus
> `"Unsupported input pixel format"`, `"Unsupported input codec"`,
> `"No keyframes in input"`, `"Error initializing filtergraph"`

Retrying a job three times (`JobRetryTime` in `internal/models/variable.go:12`,
requeued at `pkg/v1/services/job.go:334-345`) on a non-retryable error wastes ~3x GPU
time per bad upload. Classify once, fail once. Surface the last N lines of stderr in
the `CompletePlaylist` failure reason.

**29. Per-command timeout.** Every command is built with the worker's root context
(`worker.go:519`, `525`, `546`, `566`, `591`, `604`, `624`), so a wedged ffmpeg holds a
semaphore slot forever. The ladder already carries an unused `TimeoutRatio`
(`internal/models/quality_config.go:117-128`), and the legacy path has
`Video.GetTimeout()` (`internal/models/video.go:43-50`). Derive
`timeout = duration * TimeoutRatio` and use `context.WithTimeout`.

---

### P8 - Rate control (quality/bandwidth correctness)

**30. Every `-b:v` / `-maxrate` / `-bufsize` is commented out**
(`worker.go:648-652`, `698-709`, `724-728`, `778-788`), GPU uses `-qp` and CPU uses
`-crf` - two different rate-control regimes for the "same" rendition. With uncapped
CRF, a grainy source produces a 40 Mbit "720p" rung, and the `BANDWIDTH` attribute in
the HLS master playlist becomes a lie, breaking ABR.

Add capped CRF using the ladder values that already exist
(`Bitrate`/`MaxBitrate` in `internal/models/quality_config.go:7-110`):

- CPU: `-crf <Crf> -maxrate <MaxBitrate> -bufsize <2x MaxBitrate>`
- NVENC: `-rc vbr -cq <Crf> -maxrate <MaxBitrate> -bufsize <2x>`

---

### P9 - Capability discovery and server-side validation

**31. Encoder capabilities are hand-declared, never verified.** `EncoderConfig` is
loaded from a hand-edited `config.json` (`cmd/worker/main.go:36-72`), is `gorm:"-"`,
and `HeartbeatWorker` sends only id/name/driver/cuda (`cmd/worker/main.go:66-71`,
`internal/core/server/server.go:247-255`). The scheduler therefore cannot know a card
lacks hevc_nvenc 10-bit. Add a startup probe - `ffmpeg -hide_banner -encoders`,
`nvidia-smi`, plus a one-frame smoke encode per (codec, pix_fmt) pair - and report the
result in the heartbeat. Also account for the NVENC concurrent-session limit
(2 on consumer cards); `semaphore.NewWeighted(3)` at `worker.go:78` is unrelated to it.

**32. Implement `PlaylistConfig.IsValid()`** (`internal/models/input.go:18-21`, today
`return true`): resolution key in the ladder, codec in `{h264,h265}`, `SegmentType` in
`{hls,dash,mp4}`, `ContainerType` valid *for that* `SegmentType`, `Duration > 0`, sane
audio channels/sample rate, crop within source dimensions. Rejecting at `RegisterJob`
(`internal/core/server/server.go:83-124`) saves a 100 GB download before failing.

---

## Files to modify

| File | Change |
|---|---|
| `internal/core/worker/worker.go` | probe call + validation gate; fix `-map` specifiers; 4-way accel matrix; pix_fmt/colour/rotation/SAR args; fps + GOP args; capped rate control; audio normalisation; nil guard at `:311` |
| `internal/models/job.go` | `GenerateSegmentConfig` muxer-option fixes; `VideoCropInfo.Validate` against source dims; `SilenceDuration` proto alignment |
| `internal/models/quality_config.go` | `GetVideoResolutionConfig` div-by-zero + portrait-branch fix; even/clamp expressions; encoder size limits |
| `internal/models/worker.go` | `EncoderConfig.IsSupported` constant mismatch, bit-depth fields, codec-name normalisation |
| `internal/models/quality.go` | `Playlist.Exec` error classification, stderr capture, per-command timeout |
| `internal/models/input.go` | implement `IsValid()` |
| `internal/utils/m3u8_helper/helper.go` | `SetVersion(7)`; real RFC6381 CODECS; matching audio group ids |
| `internal/utils/codec/utils.go` | `hvc1` prefix; derive profile/level from the rung, not globals |
| `internal/models/stream.go` | reuse as the ffprobe unmarshal target (no new structs) |
| `internal/proto/job.proto` | new `VideoConfig`/`AudioConfig` fields (pix_fmt, colour, rotation, SAR, frame rates) |
| `cmd/worker/main.go` | encoder capability probe at startup; report in heartbeat |
| `Dockerfile.worker`, `install_ffmpeg.sh` | decoder coverage; install Bento4 or drop it |

## Verification

End-to-end, against a real worker - not unit tests on arg strings alone.

1. **Build a fixture corpus.** Start by lifting LPMS's existing pathological corpus at
   `/home/tuan/work/depin-workspace/lpms/data/` - these are real production failure
   cases, not synthetic:
   `zero-frame.ts` (audio-only), `kryp-1.ts` / `kryp-2.ts` (one keyframe / zero
   keyframes), `duplicate-audio-dts.ts`, `missing-dts.ts`,
   `missing-sei-and-pes.ts`, `broken-h264-parser.ts`, `bad-cuvid.ts`,
   `portrait.ts`, `vertical-sample.ts`, `audio.mp3`, `audio.ogg`.

   Then generate the rest with ffmpeg, committed as a generator script rather than
   as blobs:
   - 10-bit HEVC (`-pix_fmt yuv420p10le`)
   - HDR10 (`-color_primaries bt2020 -color_trc smpte2084 -colorspace bt2020nc`)
   - 4:2:2 ProRes, 4:4:4 h264
   - rotated MOV (`-metadata:s:v rotate=90`)
   - anamorphic (`-vf setsar=4/3`, 1440x1080)
   - VFR (`-vsync vfr` from a variable-rate source)
   - MKV with video on stream 3 and 5.1 audio
   - audio-only m4a; video-only mp4; zero-video-frame mp4
   - 240fps clip; odd dimensions (`641x361`)
   - AV1 and VP9 sources
   - RGB source (`-pix_fmt rgb24` AVI, or a PNG sequence)
   - truncated/corrupt file
2. **Run each through the real job path**, not `generateCmd` in isolation:
   `RegisterJob` -> `UploadMediaResource` -> worker `GetJob` -> transcode -> upload.
   `dryRun` mode (`worker.go:310-317`) is useful for snapshotting the generated args,
   but it must not be the only check - it never runs ffmpeg.
3. **Assert on output, not exit code:** `ffprobe` each rendition for expected
   resolution, pix_fmt, rotation applied, duration within tolerance of the source,
   and audio present/absent as declared. Verify segment durations are within ~5% of
   `SegmentConfig.Duration` and that all renditions share keyframe positions (the ABR
   alignment check).
4. **Force the fallback paths:** run once with the GPU present and once with it
   disabled; assert identical output geometry and comparable duration from both. Then
   run a deliberately unsupported combination and assert the error is classified
   non-retryable and the job is *not* requeued three times.
5. **Regression gate:** keep the corpus in CI with the fully-software path so it runs
   without a GPU runner.
6. **Player conformance for HEVC:** run Apple's `mediastreamvalidator` against the
   merged job-root `master.m3u8` (not the per-playlist output), and play one h265
   ladder end-to-end in Safari/iOS. Do this for both the GPU-produced and
   CPU-produced variants - today they differ.

## Further reading

`/home/tuan/work/depin-workspace/lpms/doc/quirks.md` is the highest-value document in
either repo - five production failure modes with problem/solution and upstream issue
links: audio-only segments, tiny-segment NVDEC flushing, out-of-order frames (with an
ASCII filtergraph diagram), HW session reuse, and mid-stream audio appearance. Read it
before starting P1 and P4.

Two of its findings are live-streaming-specific and should **not** be ported to VOD:
the 300 s duration cap, and the session-reuse / sentinel-packet machinery (which
exists only because CUDA init cost is comparable to a 2-second segment). Single-file
VOD jobs pay init once.
