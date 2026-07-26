---
type: fact
tags: [go-sdk, storage, uplinksdk, purego, depin]
created: 2026-07-09
agent: main
---

Implemented `GoSdkHelper` (`internal/utils/storage/gosdk.go`) — a second `StorageHelper` impl
backed by local go-sdk (`~/work/depin-workspace/go-sdk`, module `aioz-depin/go-sdk`, package
`uplinksdk`), per plan `projects/aioz-stream/plans/2026-07-09-gosdk-storagehelper.md`.

**Blocker resolved:** `vo.UUID` (needed to round-trip `Object.Id` string back into
`DownloadFile`/`CreateDownloadTicket`) lives in go-sdk's `internal/vo` package, unreachable from
aioz-stream. Added an exported `uplinksdk.ParseUUID(s string) (vo.UUID, error)` wrapper directly
in go-sdk (`~/work/depin-workspace/go-sdk/uuid.go`) — the plan explicitly anticipated this
resolution path ("may need a tiny exported helper added to go-sdk").

**Toolchain gotcha:** go-sdk's go.mod requires `go >= 1.25`; aioz-stream was on `go 1.22`
(local `go` binary is 1.23.2). `go mod tidy` auto-switched via `GOTOOLCHAIN=auto` to a cached
`go1.25.12` toolchain under `~/go/pkg/mod/golang.org/toolchain@...` and bumped aioz-stream's
go.mod to `go 1.25` — no manual toolchain install needed.

**Build tag:** go-sdk requires `-tags purego` (missing AMD64 asm in `pkg/infectious`). Added the
tag to `Makefile`'s `build`/`build-grpc` targets. CI (`.gitlab-ci.yml`) just runs `make build`,
so no separate CI edit needed. Dockerfiles only copy the prebuilt binary, no go build step there.

**Pre-existing unrelated build break:** `seeds/seed.go:34` fails to compile
(`models.DoneStatus` string used where `[]string` expected) — present before this change,
unrelated to storage work. Excluded via `go list ./... | grep -v '/seeds$'` when verifying full
build.

Verification done: `go build -tags purego`, `go vet`, and a table unit test
(`internal/utils/storage/gosdk_test.go`) asserting unsupported methods (`Delete`,
`GetAIOZPrice`, `GetTranscodeStatus`, `GetZipHeader`, `GetFileRecord`, `Transcode`) return
`ErrNotImplemented`. End-to-end upload/download round-trip against a live coord was NOT run
(no coord/identity/piece-key available in this environment) — flagged as open per plan.

**2026-07-09 identity generation:** used `~/work/depin-workspace/depin/bin/keytool` to generate a
local test client identity for the SDK: `keytool create uplink --identity-dir <dir> --difficulty 20`
(low difficulty for fast local mining; prod should use the default 36) writes
`identity.cert`/`identity.key`/`ca.cert`/`ca.key` under `<dir>/uplink/`, then
`keytool new-piece --cert-path .../identity.cert --key-path .../identity.key --out .../piece_key.json`
writes the Ed25519 piece signing key.

**Bug found + fixed in go-sdk:** `client.go`'s `loadPieceKeyIfPresent` unmarshalled
`piece_key.json` with struct tag `json:"key"`, but `keytool`'s actual output field (confirmed in
`~/work/depin-workspace/depin/cmd/keytool/cmd_piece_key.go:16`) is `"private_key"` — the doc
comment directly above the buggy code already said `private_key`, so it was a pure tag typo. Any
keytool-generated `piece_key.json` silently decoded to an empty key and failed `New()` with
"invalid private key size". Fixed the tag to `json:"private_key"` and corrected the same typo in
`README.md` (also fixed the byte count in the README, 65 not 64 — the hex string is 130 chars).
Verified fix by loading the generated identity dir through `uplinksdk.New()` — no error.

**2026-07-09 follow-up:** go-sdk commit `d93d3bd` ("feat: return upload encrypted size") changed
`Client.UploadFile` signature to `(fileID vo.UUID, encryptedSize int64, err error)` — it now
sums `upload.EncryptedSize()` across segments (post client-side AES-GCM + erasure coding, the
actual on-wire/on-disk size). Updated all 4 `UploadFile` call sites in
`internal/utils/storage/gosdk.go` (`Upload`, `PackUploadByte`, `Uploads`, `UploadRaw`) to set
`Object.Size = encryptedSize` instead of the plaintext buffer/reader length — matches what's
actually stored on DePIN, not what the caller uploaded. `go mod tidy` re-run after the go-sdk
bump; build/vet/test all green.

**2026-07-10 coord connectivity resolved.** The identity from 2026-07-09 (node ID
`325cd828-694b-e64a-6d64-e5e784a00001`, difficulty 21) now lives at
`aioz-stream/secrets/gosdk-identity/uplink/` (gitignored via the repo's `secret*` pattern — see
[[local-dev-env]]). User supplied a real, live coord peer URL:
`8ce76c37-fa80-956c-d179-c0db2a000001@tcp:68.183.189.51:7777;` — verified by a throwaway
`uplinksdk.New(...)` program (built, ran, deleted, not committed): dial succeeds. The
`kind-coord` k8s cluster running locally (`docker ps` shows `coord-control-plane`/`coord-db`,
6-day-old kind cluster) is a red herring — it's bare kube-system + local-path-storage, no actual
coord workload deployed in it; the real coord is the remote `68.183.189.51:7777` above.

**Gotcha: `WithPieceKey`/`GOSDK_PIECE_KEY_PATH` takes a DIRECTORY, not a file path**, despite the
option name. `client.go`'s `New()` does
`pieceKeyDir := identityDir; if pieceKeyPath != "" { pieceKeyDir = pieceKeyPath }` then
`loadPieceKeyIfPresent(pieceKeyDir)`, which internally joins the fixed filename
`piece_key.json` onto whatever dir you pass. Passing a path ending in `piece_key.json` itself
would double-join and fail to find the file.

Wired into both `aioz-stream/app.env` and `debug.env`:
`STORAGE_BACKEND=gosdk`, `GOSDK_IDENTITY_DIR=./secrets/gosdk-identity/uplink`,
`GOSDK_PIECE_KEY_PATH=./secrets/gosdk-identity/uplink` (same dir, per the gotcha above),
`GOSDK_COORD_PEER_URL=<the url above>`, `GOSDK_LINK_ENDPOINT=http://gosdk-link-endpoint`
(placeholder — not eagerly validated at construction, only used when building playback links).
Full end-to-end boot of the `api` binary through this path is still blocked by the unrelated
Slack `AuthTest()` panic in `cmd/http/init.go` (see [[local-dev-env]]) — storage-backend init
happens later in `init()`, so it's never reached in a real run yet; only verified in isolation.

**2026-07-10 default contract ID added.** aioz-stream's `GoSdkHelper` left `UploadParams.ContractID`
zero on every call — go-sdk auto-generates a fresh contract per upload when zero (see doc comment
on `upload.go`'s `UploadParams.ContractID`), unlike the sibling port in
`aioz-stream-core/internal/utils/cdn/gosdk.go` (a separate, independently-built repo — the
job-server/worker binaries, aka w3streamcore) which already provisions ONE contract at
construction (`CreateContractWithPlacement(ctx, "global", 999)`) and reuses it for every upload.
User wanted the same "default contract id" behavior in aioz-stream, so ported it: added
`contractRegion="global"`/`contractPlacementID=999` consts, a `contractID string` field on
`GoSdkHelper`, a `CreateContractWithPlacement` call in `MustNewGoSdkHelper` (panics on error, same
style as the existing client-init panic), and a `newUploadParams(filename, size)` helper that
`uplinksdk.ParseUUID`s the stored contractID back and builds `UploadParams` with it - wired into
all 4 upload call sites (`Upload`, `PackUploadByte`, `Uploads`, `UploadRaw`), preserving
aioz-stream's ctx-per-call style (unlike cdn's no-ctx/`context.Background()` style - that
difference is this repo's own interface convention, intentionally NOT copied over).
Verified live: build + vet + test green, and a throwaway program actually uploaded a file through
the new code path against the real coord (`68.183.189.51:7777`) - `OK: uploaded id=... size=256`
- then deleted, tree left clean.

**Found mid-session, NOT mine:** while checking `git status` after this edit, three unrelated
files showed uncommitted changes I did not make: `internal/utils/message/message.go` (the Slack
`AuthTest()` call is commented out - directly undoes the "leave it blocked for now" decision from
earlier this session, see [[local-dev-env]]), `docker-compose.yml` (new `fe` frontend service
added), and `pkg/v1/services/video.service.go` (gofmt/golines-style reformatting plus one real
fix: a missing `if err != nil { return err }` after a thumbnail-helper call in `handlePlaylist`).
Someone else (the user in another window, most likely) is actively editing this repo concurrently
- flagged to the user rather than touched.

**2026-07-10 go-sdk pinned at `c018645` ("refactor(download): return base64-encoded ticket
string") broke `GetLink`.** User is actively developing go-sdk concurrently (same pattern as
above - external edits appearing mid-session, confirmed intentional). At this commit,
`Client.CreateDownloadTicket` (`download.go`) changed its `ticket` return from raw `[]byte` to an
already-`base64.RawURLEncoding`-encoded `string` (previously the caller had to encode it). Someone
had also independently edited aioz-stream's `GetLink` (link format changed from
`%s/file/%s?ticket=...` to `%s/download?ticket=...`, dropping `object.Id` from the URL - left
as-is, not reverted) but left the old `base64.URLEncoding.EncodeToString(ticket)` call in place,
which no longer compiles once `ticket` is a `string` (`cannot use ticket (variable of type
string) as []byte value`). Reproduced the exact compile error, then fixed by just passing `ticket`
straight into the `fmt.Sprintf` (it's pre-encoded now) - the now-unused `encoding/base64` import
was auto-dropped by the project's format-on-save hook. Verified live end-to-end: upload + `GetLink`
against the real coord (`68.183.189.51:7777`) both succeed, producing a working
`.../download?ticket=<token>&expire=<unix>` URL - throwaway program deleted after.

**Takeaway:** go-sdk is a fast-moving local dependency shared across repos (aioz-stream,
aioz-stream-core) - when told "use go-sdk at `<short-hash>`", check `git log -1 --oneline` in
`~/work/depin-workspace/go-sdk` first (it may already be HEAD), then actually try building the
consuming package rather than assuming prior integration code still matches - signatures change
commit-to-commit (see the `d93d3bd` encryptedSize change above for the same pattern).

**2026-07-13 go-sdk commit `c059e8f` ("feat(upload): add system headers and user
metadata")** added `SystemHeaders`/`UserMetadata map[string]string` to
`UploadParams`, wired into both `CreateFile` and `UploadFile`'s `CreateFileRequest`.
This closed the last gap in an already-built end-to-end feature: coord (`depin` repo,
`coord/file/metadata.go`) already validated/persisted/returned system headers, and
go-sdk's download side already read them back via `WithHeaderSink` - only the upload
client wrapper hadn't been forwarding them. Landed while `aioz-stream-core` work was
mid-plan for exactly this (see [[gosdk-system-headers-upload]] in that project's
memory folder) - another instance of the user editing go-sdk concurrently in another
window.

**2026-07-14 go-sdk commit `9943ecb` ("feat(upload): parallelize piece uploads with
concurrency bounds")** converted `internal/pieceupload/manager.go`'s sequential
piece-upload loop to bounded-concurrency (semaphore, default 8, new
`Manager.SetConcurrency(n)` setter). This is purely internal to go-sdk - no new
`ClientOption` (unlike the download side's public `WithDownloadConcurrency` in
`options.go`), and `SetConcurrency` is never actually called anywhere
(`internal/segmentupload/segment.go`'s two `pieceupload.NewManager(...)` call sites
don't set it), so it's a dead unwired knob for now. Net effect for both aioz-stream
and aioz-stream-core: automatic drop-in upload speedup via the local `go.mod` replace,
zero caller-side changes needed - confirmed by rebuilding/testing aioz-stream-core
against this commit, all green. Neither repo currently overrides
`WithDownloadConcurrency` either, so there's no established pattern here for exposing
upload concurrency from the caller side - don't add one speculatively; only wire it up
if the user asks to actually tune it.

**2026-07-13 registered the uplink client with coord.** go-sdk's `account.go` has
`Client.RegisterAccount(ctx, email) (*clientpb.Client, error)` - registers the signer's public key
under an email; requires `priv_key.json` in the identity dir (loaded as `c.signer`, optional -
`RegisterAccount` errors clearly if absent). Also noticed the identity dir
(`aioz-stream/secrets/gosdk-identity/uplink/`) had been regenerated externally since the
2026-07-09 keytool mining: `ca.cert`/`ca.key` gone, replaced by OpenSSL-style
`ca.cnf`/`ca.crt.pem`/`leaf.csr.pem`/`leaf.crt.pem`/etc. (looks like a CA-signing-service /
`keytool authorize` flow rather than self-mined difficulty), plus a fresh `priv_key.json` already
present - none of it touched, `identity.cert`/`identity.key`/`piece_key.json` still work fine
as-is with `uplinksdk.New()`.

Ran a throwaway program: `GetAccount` first (confirmed `NotFound` - not yet registered), asked the
user which email to register under (auto mode's guardrail correctly blocked a first attempt that
used the user's on-file email without explicit per-action confirmation - registering against a
live external network is a real-world transaction, not assumable from a vague "register client"
instruction) - user confirmed `premium.accounts@aioz.io`. `RegisterAccount` succeeded: client id
`bb010caa-7996-63de-3822-9d9800548d01`. Re-ran `GetAccount` afterward to confirm it now returns the
same record instead of `NotFound` - registration persisted coord-side. Throwaway program deleted.

**2026-07-14 ported "system tag for file" (SystemHeaders) into aioz-stream, mirroring
`aioz-stream-core/internal/utils/cdn/gosdk.go`** (which already threads a
`systemHeaders map[string]string` param through its Upload family, wired to go-sdk's
`c059e8f` `UploadParams.SystemHeaders`). Unlike the earlier contract-ID port, this one
IS an interface-breaking change (the reference repo's `StorageHelper` interface itself
grew the param) - scoped it by first mapping the full blast radius with an Explore
agent before touching anything: `StorageHelper` (`internal/utils/storage/storage.go`)
has 5 upload methods (`Upload`, `PackUploadByte`, `Uploads`, `UploadZip`, `UploadRaw`)
across 2 implementations (`CdnHelper`, `GoSdkHelper` - no mock/test double), but only
`Upload`/`Uploads` have live callers (9 total: `UploadZip`/`UploadRaw`/`PackUploadByte`
are dead interface methods with zero call sites repo-wide).

Added `systemHeaders map[string]string` to all 5 interface methods + both
implementations. `GoSdkHelper` actually forwards it into `UploadParams.SystemHeaders`
(the real feature). `CdnHelper` (legacy `/packUpload` multipart backend) just accepts
and ignores it - that protocol has no per-file header concept, commented as such on
each method. At the 9 call sites, passed a real value only where one was already
knowable without inventing anything: `text/vtt` (`"text/" + models.CaptionFormat`) for
all 3 caption uploads (`video_caption.service.go` x2, `video.service.go`'s live-stream
caption upload) plus `media.Mimetype` for the raw media chunk upload
(`video.service.go:3448`, `MediaService.UpdateMediasStatus`). Everywhere else
(watermark, chapter, player-theme logo, thumbnail resolutions) passed `nil` rather than
guess a content-type - `SystemHeaders` is optional/nil-safe on the go-sdk side.

Verified: full repo build (`./internal/... ./cmd/... ./pkg/...`) + `go vet` +
`storage` package tests all green, plus a live throwaway upload through
`GoSdkHelper.Upload` with `{"content-type": "text/vtt"}` against the real coord -
succeeded, `id=019f5eb4-...`. Deleted after, tree left clean (only the intended 9
files + 3 unrelated-and-not-mine files from the concurrent session showed as modified).

**2026-07-14 playlist segments now embed direct storage links, skipping our own
redirect endpoint entirely.** User: "for content and thumbnail we can stop using
redirect, we can return gosdk link itself." Asked which of two shapes they meant
(controller returns the link as response body vs. playlist/thumbnail generation
embeds the real link upfront and bypasses our endpoint) - picked the latter.

Implemented for **content/playlist** in `MediaService.GetMediaContent`'s `.m3u8`
branch (`pkg/v1/services/video.service.go`): previously the segment-URI rewrite loop
just prefixed the raw playlist's embedded `segment_NNN.ts?range=...&index=...&type=...`
text with our own `/api/media/vod/<qualityId>/` (`models.SegmentUrlFormat`), so every
segment playback hit our `GetMediaContent` endpoint which then 307-redirected to the
real CDN/DePIN link - 2 round trips per segment. New `resolveSegmentURI` closure
parses the `index` out of each segment's (and `playlistData.Map`'s, if present)
embedded query string via `net/url`, looks up the matching `CdnFile` in
`quality.Files` by `(segmentFileType, index)` - same lookup `GetMediaContent` already
does for direct segment requests - then calls `storageHelper.GetLink` immediately and
writes the *real* URL straight into `segment.URI`. Falls back to the old proxy-URL
construction (plus the `.ts`/`.m4s` extension-append hack) only if `GetLink` returns
`""`, which `CdnHelper` legitimately does when `canGeneratePresignedLink` is false -
same "no fallback needed for gosdk, only for legacy CDN's edge case" pattern seen
elsewhere in this file. Net effect: player now talks directly to
`edgeserver1.zvault.ai` per segment, our API is only hit once for the manifest itself.

**Thumbnail was explicitly left alone pending another decision.** Unlike playlist,
`GetMediaThumbnail`/`Media.GetThumbnailUrl()` split across two layers: the cheap
synchronous model-method (`internal/models/video.go:573`, used in list/detail
responses) already just points at *our own* `/api/media/{id}/thumbnail` endpoint - it's
lazy, only fetched when a client actually renders that image, so there's no N+1 risk
there today. `GetMediaThumbnail` (the service method, `video.service.go:1662`)
*already* calls `storageHelper.GetLink` and only 307-redirects with the result - same
one-hop-away-from-direct shape as content used to have. The open question I flagged
back to the user rather than guess: browsers/`<img src>` follow 307 redirects
transparently, so switching that endpoint's response from "redirect to bytes" to
"200 + URL as JSON/text body" is a breaking contract change for any consumer treating
this as a direct image URL - didn't touch it without confirming the frontend actually
expects that.

**2026-07-14 follow-up: user picked the model-layer route instead** - "now return
thumbnail url in VideoObject with go-sdk link" (VideoObject == `models.MediaObject`,
`internal/models/video.go`). `NewMediaObject`/`NewMediaObjects` were pure synchronous
constructors (no ctx, no storageHelper) - used by 2 single-media-detail endpoints AND
2 list endpoints (`NewMediaObjects` in `video.controller.go`/`statistic.controller.go`),
so making `ThumbnailUrl` call `storageHelper.GetLink` means one real coord/CDN round
trip per media item in every list response. Flagged that N+1 cost explicitly before
touching anything; user said "direct link everywhere, accept per-item cost."

Full blast radius ended up bigger than just `MediaObject` - `ConvertLiveStreamMediaToResponse`/
`ConvertLiveStreamMediasToResponse` (`internal/models/live_stream_video.go`) call
`NewMediaObject` internally too, so their signatures needed the same `ctx`/`storageHelper`
threaded through as pure plumbing (their own separate, pre-existing `LiveStreamMediaResponse.Assets.ThumbnailUrl`
proxy-URL construction was deliberately left untouched - out of scope, different response
type, user only asked about `VideoObject`/`MediaObject`). Net edit: added a
`resolveThumbnailUrl(ctx, storageHelper, media, fallbackUrl)` helper in `video.go` (looks
up the "original" `ThumbnailResolution`, calls `GetLink` with `media.MediaThumbnail.Thumbnail.File.FileId`
+ that resolution's `Offset`/`Size` - mirrors `GetMediaThumbnail`'s exact lookup - falls
back to the old proxy URL on empty link or error, never fails the whole response over a
bad thumbnail link) - wired into all 3 `ThumbnailUrl` assignment sites in `NewMediaObject`.
Then threaded `ctx context.Context, storageHelper storage.StorageHelper` through: both
`video.go` functions, both `live_stream_video.go` functions, and gave `MediaController`/
`StatisticController`/`LiveStreamController` a new `storageHelper` field each (passed in
from `cmd/http/init.go`, which already had the `storageHelper` var in scope) - updated
all ~15 call sites across `video.controller.go` (5), `statistic.controller.go` (1), and
`live_stream.controller.go` (6), plus the 3 controller constructor calls in `init.go`.

Verified: full repo build/vet green (only pre-existing, not-mine noise: `limit_rate.go`
unreachable-code vet warning, and an unrelated network-dependent test failure in
`w3stream_client` trying to dial a real prod host). Live-verified via a second scratch
`api` binary (port 3098, separate from both the user's live server on 3000 and the
earlier scratch on 3099) hitting `/api/media/{id}/player.json`
(`MediaController.GetMediaObject`, no-auth route) for the same test media used in the
playlist verification - `assets.thumbnail_url` came back as a direct
`edgeserver1.zvault.ai/download?ticket=...` link, confirmed it resolves (200, 50KB).
Scratch binary/env deleted after.

**Not done, worth a future look:** the thumbnail batching in `internal/utils/image/image.go`'s
`GenerateThumbnail` has the same "first-object-id-captures-the-whole-batch" shape as the
HLS segment-batching bug found earlier (`aioz-stream-core/internal/core/general_hanlder.go`)
- `media.MediaThumbnail.Thumbnail.File.FileId` is set from only the *first* uploaded
resolution's object id, then every resolution's `GetLink` call reuses that one Id with a
per-resolution Offset/Size that (for the gosdk backend) doesn't actually address into a
shared blob. Not confirmed broken (didn't reproduce it - "original" resolution seems to
usually be uploaded last/alone in the observed test data so it happened to resolve
correctly), and fixing it wasn't part of this task - just inherited the existing lookup
pattern as-is from `GetMediaThumbnail`. Someone should verify non-"original" resolutions
actually resolve to the right image bytes under the gosdk backend.

**2026-07-14 same-day panic in production from this change, fixed.** User pasted a stack
trace: panic inside `internal/controllers.(*MediaController).GetMediaList` pointing at
`video.go:497` - exactly the `NewMediaObject(ctx, storageHelper, media)` call inside
`NewMediaObjects`. Root cause (found by reading code, no need to re-reproduce over HTTP -
the evidence was conclusive): `resolveThumbnailUrl` unconditionally dereferenced
`media.MediaThumbnail.Thumbnail.Resolutions`, but `Thumbnail *Thumbnail` is only populated
when a query does the FULL 4-level preload chain (`MediaThumbnail` →
`MediaThumbnail.Thumbnail` → `.Resolutions` → `.File`). Grepped every
`Preload("MediaThumbnail"...)` in `pkg/v1/repositories/`: `GetMediaById` and
`GetUserMediaById` (single-detail paths, used by the 3 `NewMediaObject` direct call
sites) already had the full chain and were fine. `GetUserMedias`
(`video.repository.go:352`, backs `GetMediaList`) and `GetStatisticMedias`
(`statistic.repository.go:1301`) only did the shallow `Preload("MediaThumbnail")` -
both feed `NewMediaObjects` (list responses), both would nil-pointer-panic for ANY media
with a thumbnail once list views started actually reaching the new thumbnail-resolution
code path.

Two-part fix (root cause, not just the crash): (1) added
`if media.MediaThumbnail.Thumbnail == nil { return fallbackUrl }` and a
`media.MediaThumbnail.Thumbnail.File == nil` check in `resolveThumbnailUrl` - defensive,
correct regardless of what any given repo query preloads, degrades to the old proxy URL
instead of crashing; (2) added the missing
`Preload("MediaThumbnail.Thumbnail")`/`.Resolutions`/`.File` to both `GetUserMedias` and
`GetStatisticMedias` so list views actually GET a resolvable thumbnail instead of silently
falling back forever - otherwise the defensive fix alone would've quietly defeated the
whole point of "direct link everywhere" for every list endpoint. (Live-stream media
queries in `live_stream_video.repository.go` don't preload `MediaThumbnail` at all in most
paths, but `NewMediaObject`'s existing `if media.MediaThumbnail != nil` outer guard already
covers that - not a new bug, left alone.)

Verified via a throwaway program that called `mediaRepo.GetUserMedias` +
`models.NewMediaObjects` directly (same code path `GetMediaList` uses, bypassing the need
for a JWT to hit it over HTTP) against the real dev DB (user id
`c86216a4-6529-4219-9ad4-c58b33dfd0b3`, 10 media incl. one with a thumbnail) - no panic,
all 10 got a real resolved `thumbnail_url` (confirming the preload fix, not just the nil
guard). Deleted after.

**2026-07-14 same-day, second incident: real prod 500 from the playlist direct-link
change.** User pasted a live error log (real external IP, trace_id, `/playlist.m3u8`
500): `get segment link: gosdk: get link: uplinksdk: get download ticket: rpc error:
code = Canceled desc = context canceled while waiting for connections to become ready`.

Root cause: the playlist `resolveSegmentURI` change (earlier today) turned what used to
be a cheap text-rewrite loop into up to N sequential coord RPC calls - one
`storageHelper.GetLink` per segment, one at a time, all inside a single HTTP handler.
For a 53-segment video that's 53 back-to-back round trips before the response can even
start - long enough for a client/proxy timeout (or a transient reconnect) to cancel the
request context mid-resolution, which grpc-go surfaces as exactly this
"context canceled while waiting for connections to become ready" (a call landing while
the underlying ClientConn is transiently reconnecting, then the context dies before it's
ready again). This was a genuine regression introduced by today's own change, not a
pre-existing issue.

Fix (root cause, not symptom - two parts, both in `resolveSegmentURI`/its caller loop
in `video.service.go`):
1. **Resilience:** `GetLink` errors inside `resolveSegmentURI` no longer hard-fail the
   whole playlist - they now log a warning and fall through to the existing proxy-URL
   fallback (previously only the `link == ""` case degraded gracefully; now transient
   RPC errors do too). Genuine data-integrity errors (bad URI, unparseable index,
   missing `CdnFile` for that index) still hard-fail, since those indicate real bugs
   worth surfacing, not something to paper over.
2. **Latency:** the segment-resolution loop is now bounded-concurrent (`sync.WaitGroup`
   + a `chan struct{}` semaphore, cap 8 - matches go-sdk's own internal piece-upload
   default concurrency) instead of one-at-a-time, cutting wall-clock roughly from
   sum-of-53-round-trips to slowest-single-round-trip. `playlistData.Map` (fmp4 init
   segment, only one such call) stays single/sequential - nothing to parallelize there.

Verified: full build/vet clean. Live-timed on a third scratch `api` binary (port 3097) -
the same 53-segment quality (`e3461e97-...`) that previously would've taken tens of
seconds sequentially now resolves in **1.3s total**, all 53 segments still correctly
resolved to distinct direct `edgeserver1.zvault.ai` links. Scratch binary/env deleted
after.

**Takeaway for next time a service method starts doing per-item network calls in a
loop:** always ask "does this scale with a user-controlled count" (segment count here,
media-list length for the thumbnail change above) - if yes, bound the concurrency AND
make per-item failures degrade rather than abort the whole batch, from the start rather
than after a page-1 production incident.

**2026-07-26 added a local-disk cache for the RAW playlist.m3u8 bytes, video-only
renditions of videos >20min only.** Continuation of the segment-resolution-latency
thread above. User's ask ("save the playlist.m3u8 ... at playlist.m3u8") could've meant
several things - clarified via AskUserQuestion before touching code, since the naive
reading (cache the fully segment-resolved playlist) directly conflicts with
`GoSdkHelper.GetLink`'s `GetLinkExpiry = 24h` presigned-ticket lifetime: baking resolved
links into a long-lived cached file would silently break playback >24h later. Presented
that conflict + two options; user picked the narrow one - **cache only the raw
pre-rewrite playlist bytes** (skips the one `storageHelper.Download` RPC per request),
still resolve every segment's link live every request (no staleness risk introduced).

Implementation (`pkg/v1/services/video.service.go`'s `GetMediaContent`, `else` branch
~line 1892 handling `CdnVideoPlaylistType`/`CdnAudioPlaylistType`): gated on
`fileType == models.CdnVideoPlaylistType &amp;&amp; quality.Media.GetMediaDuration() >
20*time.Minute` (audio-only and short videos untouched, same per-request Download as
always). Cache path: `filepath.Join(s.outputPath, quality.MediaId, quality.Id,
"playlist.m3u8")` - reuses the existing `s.outputPath`/`os.WriteFile`/`filepath.Join`
convention already used for captions elsewhere in this file (`s.outputPath` already a
`MediaService` field, no new config needed). `os.ReadFile` on the cache path first; any
error (including not-exist) falls through to the normal `storageHelper.Download` +
`os.MkdirAll`/`os.WriteFile` (best-effort, `slog.WarnContext` on failure, never fails
the request - matches the existing "degrade, don't abort" pattern in this same
function's `resolveSegmentURI`).

Needed `Preload("Media.Format")` added to `QualityRepository.GetQualityById`
(`pkg/v1/repositories/quality.repository.go`) - duration lives on `Media.Format.Duration`
(string secs, via `Media.GetMediaDuration()`), and the existing preload set
(`Media`/`Files`/`Files.File`) didn't include it, so duration was always 0 in this code
path before this change.

**No shared-storage risk to worry about (checked before implementing):** this
docker-compose only ever runs ONE `api` container (no horizontal replica config for it),
so a per-instance local-disk cache doesn't have a cross-replica staleness/inconsistency
problem here - would need revisiting (shared volume or move to the storage backend
instead of local disk) if `api` is ever scaled to N replicas.

Verified: `go build`/`go vet -tags purego ./cmd/... ./pkg/... ./internal/...` clean
(only the pre-existing, already-flagged `limit_rate.go` unreachable-code vet note, not
touched, per earlier explicit user decision to leave it alone).

**Not done / worth a future look:** no eviction or invalidation - if the source
playlist file itself ever changes (re-transcode, quality edit) the local cache would
silently keep serving the stale raw bytes forever. Not a concern raised by the user and
out of scope for this ask, but worth flagging if this pattern gets reused elsewhere.

**REVERTED same day, immediately after.** User's actual ask was literal - "download the
m3u8 for me, not edit the code". The whole caching-feature interpretation above
(clarifying questions, TTL discussion, code change) was a misread of an ambiguous
one-line request; should have asked "do you mean a code feature, or literally fetch a
file" as the FIRST clarifying question, not jumped straight to "which caching design."
Reverted both edits exactly (`Preload("Media.Format")` removed from
`QualityRepository.GetQualityById`, `GetMediaContent`'s `else` branch restored to the
original unconditional `Download`+buffer, no local-disk check) - confirmed clean via
`go build ./cmd/... ./pkg/... ./internal/...`. Treat the design notes above as dead/
unused - not reflecting current code - kept only as a record of the design work in case
a real "cache the playlist" feature gets requested later.

**Lesson: for a short, ambiguous imperative sentence, disambiguate "do this as code" vs
"do this as an action right now" BEFORE presenting design options** - the two have
almost nothing in common (one is a PR, the other is a `curl`/download command), so
picking wrong wastes a full round of clarifying questions on the wrong axis entirely.

**2026-07-24 "register new go-sdk client" turned out to be closing packaging gaps, not
core wiring** - an Explore agent confirmed the factory switch
(`cmd/http/init.go`/`cmd/grpc/init.go` picking `MustNewGoSdkHelper` vs
`MustNewCdnHelper` off `STORAGE_BACKEND`) and all the contract-id/system-header
plumbing above were already done and building clean. What was actually missing:
(1) `env-example/app.env` had zero `GOSDK_*`/`STORAGE_BACKEND` entries - the real
`app.env`/`debug.env` values only ever lived in the gitignored files, so a fresh
clone had no template to go from. Added a `STORAGE_BACKEND=cdn` (default, not gosdk -
example should stay on the safe/legacy default) + 4 `GOSDK_*` placeholder block right
after `CDN_URL`/`HUB_URL`. (2) `docker-compose.yml`'s `api`/`livestream`/`livestream2`/
`grpc` services never mounted the `secrets/gosdk-identity/uplink` dir that
`GOSDK_IDENTITY_DIR`/`GOSDK_PIECE_KEY_PATH` point at (`./secrets/gosdk-identity/uplink`,
relative, WORKDIR `/app` in every Dockerfile) - containers running the gosdk backend
would fail to load the identity even with correct env vars. Added `./secrets:/app/secrets`
to those 4 services' volume lists. Deliberately left the two known pre-existing
anomalies (`limit_rate.go` rate-limit bypass, `message.go` Slack `AuthTest()` disabled)
untouched per explicit user choice - flagged but out of scope this round.

**2026-07-24 same day, re-registered the existing identity with coord - the
2026-07-13 registration was gone.** User clarified "register client" meant the
coord-side account (`Client.RegisterAccount`), not code wiring - asked "use our
current identity" explicitly, so no new keytool identity generated. Checked live
first via a throwaway `GetAccount(ctx)` call (same identity dir
`secrets/gosdk-identity/uplink/`, same coord peer URL
`8ce76c37-...@tcp:68.183.189.51:7777;`, files untouched since 2026-07-10/17 per
mtimes) - got `NotFound`, even though this exact identity was successfully
registered on 2026-07-13 (client id `bb010caa-7996-63de-3822-9d9800548d01`). The
dev coord at `68.183.189.51` had evidently been reset/wiped server-side between
then and now - **coord-side registration does not persist across coord resets even
though the identity/keys on disk are unchanged; don't assume a past registration
memory entry still holds without a live `GetAccount` check first.**

Confirmed email via AskUserQuestion (external-network write, same guardrail as
2026-07-13) - user picked the same `premium.accounts@aioz.io` used before.
Re-ran `RegisterAccount` (throwaway program at `cmd/tmp_regcheck/main.go` inside
the repo, needed for the local go-sdk module replace to resolve - NOT placed in
`/tmp` scratchpad, which isn't part of the module) - succeeded, got back the exact
same client id `bb010caa-7996-63de-3822-9d9800548d01` (coord apparently
deterministically derives the client id from the identity's public key/address,
not a fresh random one per registration). Verified with a follow-up `GetAccount`
call in the same run - now resolves. Throwaway program deleted after, `git status`
confirmed only the pre-existing `cmd/http/init.go` diff remains.

**2026-07-24 same day, minutes later: registration gone AGAIN.** User said
"recreate client again" - clarified via AskUserQuestion it meant re-run
`RegisterAccount` on the same identity (not mine a new one). Re-ran the same
throwaway-in-`cmd/tmp_regcheck` check: `GetAccount` → `NotFound` again, despite
having just confirmed registration successfully moments earlier in this same
session. Re-registered, got the same client id `bb010caa-...` again, confirmed
live. **Pattern now repeats twice in one session** - this dev coord
(`68.183.189.51:7777`) does not durably persist `RegisterAccount` state; it's
either being restarted/redeployed very frequently or not backed by persistent
storage. Don't treat a "confirmed registered" `GetAccount` result as good for
more than the current session - always re-check live immediately before anything
that depends on registration (e.g. don't skip the check next time on the
assumption last session's confirmation still holds). Worth surfacing to whoever
owns the dev coord deployment if registration needs to actually stick.

**2026-07-24 same day, go-sdk bumped to `f9cbaee` ("feat(pool): single-use
connections for relay circuits").** User asked to "use new gosdk version at
f9cbaeeb4a4e" - checked `~/work/depin-workspace/go-sdk` first and it was
already sitting at that exact commit (one commit ahead of the `f78defc` HEAD
seen earlier this same session - `f9cbaee` is the only commit in between, so
someone, presumably the user in another window per the established concurrent-
editing pattern, had already advanced it). No checkout needed - just verified
compatibility: `go build -tags purego ./cmd/http/... ./cmd/grpc/...`,
`go vet` (same + `./internal/utils/storage/...`), and
`go test ./internal/utils/storage/...` all green, no aioz-stream-side changes
needed. The commit is purely internal to go-sdk's connection pool (relay
circuit handling), no public API surface touched.

**Noticed while running `go mod tidy` (unrelated to the version bump - don't
run this in aioz-stream going forward without scoping it):** `postgres_data/`
(the Docker Postgres data dir at repo root, recreated root:uid-999-owned after
the 2026-07-24 DB wipe above, and actually already unreadable before that too
per its original `dnsmasq:root` ownership) breaks any `go` command that walks
the whole module tree (`go mod tidy`, `go build ./...`, `go vet ./...`) with
`permission denied` - `go` resolves it as if it were an importable subpackage
of this module's own path. Not something to fix (it's Docker-internal state,
correctly gitignored) - just always use targeted package paths
(`./cmd/http/...` etc, as this whole gosdk workstream already does) instead of
whole-module `./...` patterns in this repo.
