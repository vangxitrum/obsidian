---
type: fact
tags: [aioz-stream, crop, swagger, code-review]
created: 2026-08-26
agent: main
---

Fixed 4 whole-branch review findings on the crop-box-for-transcode feature
(worktree `crop-video-transcode-input`, base commit b1b60dfb, fix commit 58cd6283):
doc-comment expansion on `CreateMediaRequest.CropInfo`, audio-media rejection of
`crop_info` in `video.controller.go`'s `CreateMediaObject`, capping the H265
rendition-ladder `maxSize` to the crop's dimensions (not full source) in
`video.service.go`'s `completeMedia`, and regenerating swagger docs.

**Gotcha:** `make gen-swagger` (`swag init --parseDependency ... && swag fmt`)
doesn't just touch `docs/*` - `swag fmt` walks the whole repo and reformats
import-grouping (stdlib vs external) in unrelated already-gofmt-clean generated
files too (saw it rewrite `internal/proto/job/job.pb.go` and `job_grpc.pb.go`,
pure import reordering, zero semantic diff). Check `git status` after running it
and revert any unrelated files (`git stash push -- <paths>` then `git stash drop`)
to keep the regen commit scoped.

Confirms [[go-edit-formatter-hook]]: used Bash/python3 in-place edits (not
Edit/Write tool) for the two Go source fixes to avoid the global PostToolUse
formatter hook re-wrapping unrelated code.
