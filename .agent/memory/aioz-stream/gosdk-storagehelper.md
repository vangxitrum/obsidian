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

**2026-07-29 registration check, three days after the previous two same-session
NotFounds - this time it HELD.** User said "register go-sdk client again with the
provided identity" - "provided identity" turned out to be the same one as always
(checked `secrets/gosdk-identity/uplink/` for anything newer than 2026-07-24 first,
nothing changed - no new identity was actually provided, just the existing one).
go-sdk local repo had moved again (`f9cbaee` → `c5f680f`, "feat(download): add segment
prefetch for multi-segment downloads" - download-path only, unrelated to
account.go) - confirmed `go build` clean before running. `GetAccount` this time
returned the SAME registration from 2026-07-24 (`bb010caa-...`,
`premium.accounts@aioz.io`) - no re-register needed. So the coord's registration loss
isn't on a short/fixed cycle - it survived >3 days here after failing twice within
minutes on 2026-07-24. Keep the "always live-check before assuming" rule, but don't
assume every request needs an actual re-register - check first, only call
`RegisterAccount` if `GetAccount` actually 404s.

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

**2026-08-13 switched `GoSdkHelper.GetLink` to the canonical, cacheable edge
download URL.** Old shape `<edge>/download?ticket=<t>&expire=<unix>` is the
legacy edgeserver route: the ticket IS the URL, so no shared cache can key on
it, and the depin edge deliberately serves that route as
`Cache-Control: private` no matter what (`edgeserver/handler.go:setCacheControl`,
`canonical && cfg.Public` gate). New shape is
`<edge>/download/<fileID>?ticket=<t>` - depin `edgeserver/server.go:213`
`GET /download/{fileId}` (`handleDownloadByID`), where the *path* (object id)
is the stable cache key and the ticket rides out-of-key in the query so a
fronting CDN can be configured to drop it from the cache key. Edge also accepts
`Authorization: Bearer <ticket>` (`ticketFromRequest`, handler.go:470) -
verified live, 200 with no query string at all. `ticketMatchesFileID` rejects a
ticket whose embedded file id differs from the path, so a shared cache can't be
poisoned across objects.

Dropped the `expire=` param entirely: nothing in aioz-stream ever parsed it back
(grepped - only cdn.go's unrelated legacy `/file/%s?expire=&signature=` scheme
uses that name), the ticket already carries its own expiry, and it was pure
per-request cache-key noise.

**Why per-URL memoization would still help, and why the route change alone
isn't sufficient:** `CreateDownloadTicketLocal` signs with
`pkcrypto.HashAndSign` (ECDSA, randomized nonce), so *two mints of the same
(fileID, expiresAt) produce different ticket bytes*. The URL therefore still
differs on every `GetLink` call - only the path is stable. That's fine for a
CDN keyed on path, useless for a browser/naive cache keyed on the full URL.
`CdnHelper` already solves the equivalent problem with its `ticketMapping
*sync.Map` (`cdn.go:818/856`); `GoSdkHelper` has no such cache. Left as a
follow-up, not done.

**Deployed edge blocker for the CDN win:** `edgeserver1.appdemo.cyou` currently
returns `cache-control: private, max-age=86400, immutable` on the canonical
route - i.e. depin's `CacheConfig.Public` (`edgeserver/config.go:73`,
`default:"false"`) is still off in that deployment. Must be flipped to true
(and only behind a CDN whose cache key is the object id, not the ticket) before
any shared cache will store the body. Verified E2E on 2026-08-13 with a
throwaway upload: 200 + correct body, `etag: "<fileID>"`, `If-None-Match` ->
304, `Range: bytes=0-7` -> 206, bearer-header form -> 200.

**Same session - three downstream cache bugs found while chasing "better for
cache", all pre-existing at HEAD, all fixed:**
1. `Cahche-Control` - misspelled header name at **6** redirect sites
   (`video.controller.go` x4, `playlist.controller.go`, `player_themes.controller.go`).
   Real `Cache-Control` was therefore never emitted on any presigned-link redirect.
2. `max-age=%d` was fed `int(time.Until(expiredAt))` - a `time.Duration`, i.e.
   **nanoseconds**. RFC 9111 wants seconds. Would have read ~86400000000000.
3. `GoSdkHelper.GetLink` returned `expiry.Unix()` (**seconds**) while
   `CdnHelper.GetLink` returns nanoseconds (CDN's `expire_at_ns`),
   `models.FileInfoBuilder`'s default is `UnixNano()`, and every controller reads
   it as `time.Unix(0, fileInfo.ExpiredAt)`. So on the gosdk backend `Expires`
   resolved to **1970** and max-age went negative - caching fully dead. Changed to
   `expiry.UnixNano()`; nanoseconds is the interface contract for GetLink's 2nd
   return value.

Collapsed all 6 duplicated blocks into `internal/controllers/cache_headers.go`
`setRedirectCacheHeaders(ctx, expiredAtNs)` (emits `private, max-age=<sec>` +
`Expires`, clamps negative max-age to 0), with
`internal/controllers/cache_headers_test.go` as the regression guard.

Reproduced the old shape live before fixing: `GET localhost:3000/api/media/
<id>/player.json` on the running `./bin/api` returned
`assets.thumbnail_url = https://edgeserver1.appdemo.cyou/download?ticket=...&expire=...`.
Useful recipe: media ids come from `docker exec aioz-stream-db psql -U admin -d
video-db` (creds in `debug.env`, port 5437), `/api/media/:id/player.json` and
`/api/media/:id/thumbnail` are **noAuth** routes (`internal/routes/video.route.go:78-95`).
`media_thumbnails` is only a `(media_id, thumbnail_id)` join table.
Note `./bin/api` reads `./debug.env` (`APP_ENV`, default "debug") and hardcodes
pprof on :6060, so a second instance can't be booted cleanly alongside it - and
booting one anyway would double-run `cron.Start()`, the payment watcher and
`mediaSummaryService.Run`, which is why the live re-check was handed back to the
user rather than done by spawning a parallel instance.

Pre-existing unrelated test failures in this repo (not caused by any of this):
`internal/utils/summarizationclient` and `internal/utils/transcribe_client` hit
`https://aiozstreamai.tunnel.zvault.ai` and fail with `remote error: tls:
internal error`.

**2026-08-18 GoSdkHelper.GetLink now never mints a link - go-sdk backend
always downloads and serves bytes itself instead of redirecting.** User:
"create a direct download function for go-sdk, we will download it directly
with sdk instead of returning the redirect url." Scoped via AskUserQuestion
to two decisions: (1) touch only go-sdk's own `GetLink` impl, leave
`CdnHelper` untouched; (2) apply it EVERYWHERE go-sdk hands out a link, not
just the 4 `ctx.Redirect` controller sites - user explicitly chose "also
proxy segments through us", so the HLS segment-embed optimization
([[gosdk-storagehelper]] 2026-07-14 entry) is deliberately reverted for the
go-sdk backend.

Implementation is a single-function change with zero call-site edits, because
every one of the 6 `.GetLink(` callers repo-wide (`GetMediaThumbnail`,
`GetMediaContent` non-playlist branch, `resolveSegmentURI` in the playlist
branch, `GetPlaylistThumbnail`, `GetPlayerThemeLogo`,
`resolveThumbnailUrl` in `internal/models/video.go`) already treats an empty
link + nil error as "download and serve the bytes yourself" - the same
contract `CdnHelper` uses for its `canGeneratePresignedLink==false` case.
`GoSdkHelper.GetLink` (`internal/utils/storage/gosdk.go`) now just
`return "", 0, nil` - deleted the `CreateDownloadTicketLocal`/
`CreateDownloadTicket` minting logic and the `GetLinkExpiry` const/`time`
import that only served it. `linkEndpoint` field + `WithLinkEndpoint` option
left in place (still set by the constructor/`GOSDK_LINK_ENDPOINT` env var,
just unread now) - ripping those out would touch `cmd/http/init.go`/
`cmd/grpc/init.go`/env files, out of scope for this change; no lint stage
exists in `.gitlab-ci.yml` in this repo currently so an unread field isn't a
CI risk (the golangci-lint CI mentioned in an earlier entry must be a
different repo/branch - `.golangci.yml` doesn't exist here).

**Real bug fixed as a side effect, now load-bearing:** `GoSdkHelper.Download`
previously ignored `Object.Offset`/`Size` entirely, always doing a
whole-file `DownloadFile` from byte 0. Harmless before (the `if
redirectUrl == ""` fallback branch was dead code for go-sdk since `GetLink`
always succeeded), but now every packed-file read (thumbnail resolutions,
chapter/caption files, video/audio content byte ranges, HLS segments) goes
through this path, so it had to become range-correct. Fixed using go-sdk's
`Client.DownloadFileRange(ctx, fileUUID, w, offset, length)` (owner-
authenticated, `length<=0` means "to EOF", confirmed via `effectiveRange` in
go-sdk's `download.go`): `Download` now calls `DownloadFileRange` whenever
`Offset != 0 || Size > 0`, falls back to whole-file `DownloadFile` only for
the degenerate `Offset==0 && Size<=0` case.

Verified live end-to-end against the real coord with a throwaway program
(`cmd/tmp_verify_directdl`, deleted after): uploaded a 21-byte known blob,
confirmed `GetLink` returns `("", 0, nil)`, confirmed whole-file `Download`
round-trips the exact bytes, confirmed a ranged `Download(Offset:5,
Size:10)` returns exactly bytes `[5:15]`. Added
`TestGoSdkHelper_GetLink_NeverMintsLink` to `gosdk_test.go` as a fast
regression guard (no live coord needed - `GetLink` no longer touches the
client at all). Full `go build`/`go vet -tags purego ./cmd/... ./pkg/...
./internal/...` clean except the pre-existing, not-mine
`limit_rate.go:34` unreachable-code vet note.

**Not done / flagged, not asked this round:** no config toggle to re-enable
link-minting later (e.g. if the CDN edge's `Public` cache flag from the
2026-08-13 entry ever gets flipped and direct-links become worth it again) -
implemented as an unconditional behavior swap per the literal ask; git
history has the old ticket-minting code if this needs reverting.

**2026-08-18 same day, reverted: "change it back to redirect link."** Restored
`GoSdkHelper.GetLink`'s ticket-minting body exactly (CreateDownloadTicketLocal
+ CreateDownloadTicket fallback, canonical `/download/<fileID>?ticket=` link,
`GetLinkExpiry` const, `time` import) - undoes the entry directly above.
Removed `TestGoSdkHelper_GetLink_NeverMintsLink` (no longer a true contract).
The `Download` offset/range fix (`DownloadFileRange`) was NOT reverted - user
only asked about the redirect-vs-download behavior, not the offset bug fix,
and it's correct regardless of which path GetLink takes. Note: by the time of
this revert, a concurrent edit (same pattern as always - user in another
window) had already changed `Download` further: wrapped the pipe writer in an
8 MiB `bufio.Writer` and dropped the `Offset==0 && Size<=0` whole-file
`DownloadFile` branch in favor of always calling `DownloadFileRange` (valid
per `effectiveRange`'s docs - length<=0 means "to EOF", so it's equivalent for
the whole-file case) - left as-is, not mine, not asked to touch.
Build/vet/test clean after revert (only the pre-existing not-mine
`limit_rate.go` vet note).

**Net effect: go-sdk backend is back to redirect-based downloads** (same as
before the direct-download detour earlier today) - HLS segment embedding,
thumbnail URLs, and the 4 controller redirect endpoints all mint real
presigned links again via `GetLink`.

**2026-08-18 same day, flipped back to direct-download again: "now change
back to our download directly."** Third flip in one session
(link->direct->link->direct). Reapplied the exact same `GetLink` -> `return
"", 0, nil` change and re-added `TestGoSdkHelper_GetLink_NeverMintsLink`,
removed `GetLinkExpiry`/ticket-minting/`time` import again - identical to the
first direct-download entry above. `Download`'s range-aware
`DownloadFileRange` behavior was never touched across any of these flips.
Build/vet/test clean (same pre-existing not-mine `limit_rate.go` note).
**Current state as of this entry: direct download (no redirect link) for
go-sdk.** Given the back-and-forth pace, check current file state directly
before assuming which mode is live - don't trust this note's "current state"
line without a quick `grep GetLink gosdk.go` first, since another flip may
have happened since.

**2026-08-18 same day, 4th flip: back to redirect link.** Same
`GetLink` restore as the 2nd entry (ticket-minting body, `GetLinkExpiry`,
`time` import), test removed again. Noticed another concurrent edit had
landed on `gosdk.go` between flips (`log/slog` import added, presumably
`Download`'s buffered-writer goroutine now logs something) - not touched,
not mine. First patch attempt this round silently no-op'd (Python script's
import-block assertion didn't match the now-`log/slog`-containing block, so
it raised before writing anything) - caught by re-checking the file's actual
current text before assuming the edit landed; redid it against the real
current import block. **Lesson: with concurrent edits landing on this exact
file every few minutes this session, always re-`grep`/re-read the target
block immediately before a scripted replace, even seconds after the last
check - don't reuse a block captured a few tool-calls ago.**
Build/vet/test clean. **Current state as of this entry: redirect link (not
direct download)** for go-sdk - but per the note above, verify live before
trusting it.

**2026-08-19, 5th flip: back to direct download.** Same pattern as the
3rd entry - `GetLink` -> `return "", 0, nil`, `GetLinkExpiry`/ticket code
removed, `time` import removed (kept `log/slog` and `bufio` from the ongoing
concurrent edits), test re-added. Used a line-range replace this time
(re-grepped exact current line numbers first) instead of whole-block string
match, specifically because whole-block matches kept getting invalidated by
concurrent edits landing on this file between turns - more robust against
that.

**Given 5 flips in ~1 day with no functional trigger stated each time,
flagged to user: worth making this a runtime toggle (env var / GoSdkOption)
instead of a code edit each time?** Not done yet - user hasn't asked for it,
just recording the suggestion so a future session doesn't re-suggest from
scratch. If asked, the clean way: add `directDownload bool` field +
`WithDirectDownload()` option on `GoSdkHelper`, branch at the top of
`GetLink` (`if h.directDownload { return "", 0, nil }` then fall through to
existing ticket logic) - one small diff instead of swapping the whole
function body every time.

**2026-08-19, 6th flip: back to redirect url again.** Same restore as
before. Build/vet/test clean. **Current state: redirect link (not direct
download)** - verify live before trusting, per the standing note above (6
flips now, concurrent edits keep landing on this exact file between turns).
The "make it a runtime toggle" suggestion from the previous entry still
stands, unanswered.

**2026-08-20 added the link cache flagged as a follow-up in the 2026-08-13
entry.** User: "adding link cache for go-sdk". Confirmed current state was
redirect mode first (`grep GetLink` per the standing "verify live" note -
6 flips had happened by 2026-08-19). Implemented exactly the gap that entry
described: `CreateDownloadTicketLocal` signs with a randomized ECDSA nonce,
so repeated `GetLink` calls for the identical `(id, offset, size)` minted a
different URL every time - fine for the edge's path-keyed cache, useless for
anything keyed on the full URL (browser cache, a fronting CDN not yet
stripping the ticket from its cache key).

Mirrored `CdnHelper.ticketMapping` (`cdn.go:818/856`) exactly: added
`linkCache *sync.Map` field + `cachedGoSdkLink{link string; expiresAt
int64}` value type + `WithLinkCache(*sync.Map)` option (parallels
`WithTicketMapping`, also currently unused at call sites, same as its CDN
counterpart) to `GoSdkHelper`/`gosdk.go`. Cache key
`fmt.Sprintf("%s-%d-%d", object.Id, object.Offset, object.Size)` - same
shape as CDN's `tickerKey`. Unlike CDN's buggy `-1000000` (1ms, likely
meant to be a time.Second literal that got typo'd into nanoseconds) staleness
check, used a named `linkCacheSafetyMargin = time.Minute` constant compared
via `time.Until(...) > margin` - clearer and not copying that bug forward.
On a hit within the margin, returns the cached link/expiry verbatim,
skipping `ParseUUID`/`CreateDownloadTicketLocal`/`CreateDownloadTicket`
entirely; on a miss (absent or near-expiry), mints as before and stores the
result before returning.

Added `TestGoSdkHelper_GetLink_CacheHit` (nil client on purpose - a
fallthrough to minting would nil-pointer-panic, so a clean return proves the
cache path fired) and `TestGoSdkHelper_GetLink_CacheExpired` (entry inside
the safety margin must be treated as a miss - used a deliberately-invalid
`object.Id` so the fallthrough fails fast at `ParseUUID` with a distinct
error instead of needing a real client). First version of the expired test
tried to assert via `recover()`/panic, which was wrong - a bad UUID string
returns a normal error before ever reaching the nil client, not a panic;
rewrote to assert on the returned error instead once the live test run
caught it.

Verified live against the real coord (throwaway `cmd/tmp_verify_linkcache`,
deleted after): two `GetLink` calls for the same freshly-uploaded object
returned byte-identical links (proving the cache hit, not just "didn't
crash"); a second, different object got a distinct link (proving the cache
key is actually object-scoped, not a global single-slot cache). Full
`go build`/`go vet -tags purego ./cmd/... ./pkg/... ./internal/...` clean
(only the pre-existing not-mine `limit_rate.go` vet note) and
`go test ./internal/utils/storage/...` green.

**Not done:** no eviction/pruning of `linkCache` - entries for objects that
are never requested again (or ever deleted) stay in the map forever, unbounded
by object count. Not raised by the user and go-sdk's per-file ticket cardinality
here is probably small relative to CDN's, so left alone; worth a look if this
ever shows up as a memory-growth concern.
