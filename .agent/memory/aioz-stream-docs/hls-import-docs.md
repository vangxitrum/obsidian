---
type: fact
tags: [docs, hls-import, m3u8, nextra]
created: 2026-09-10
agent: main
---

The HLS/M3U8 import docs live on branch `docs/import-m3u8` in aioz-stream-docs:
- `docs/pages/video-management/import-m3u8.mdx`, the guide, linked from `_meta.json`
- `## Import video from HLS URL` in `docs/pages/aioz-stream-api/media/video.mdx`

LLM docs (`docs/public/llms*`) were deliberately left untouched.

**Backend facts the docs depend on (aioz-stream):**
- There's no dedicated route: import is `POST /api/media/create` with `source_url`.
- It returns 202, although swagger says 200.
- It sits behind `HLS_IMPORT_ENABLED` (default false).
- It exists only on `origin/stag`, not `main` or `master`, as of 2026-09-10.
- Audio-group support (`ErrPartialAudioGroup`/`ErrMultipleAudioRenditions`) was uncommitted work on `feat/migrate-with-m3u8`. Committed code rejects all demuxed audio (`ErrDemuxedAudio`). Re-check the audio rules in the guide before publishing.
- Audio-only masters and SUBTITLES renditions are not importable.

Related: [[m3u8-import-vs-demuxed-audio]] (aioz-stream memory).

**Docs repo gotchas:**
- Build with `pnpm build` in `docs/` (next build).
- The `ApiDocumentation` component (`docs/components/swagger`) takes `body`, `headers` and `parameters` arrays. Use `isSeparator` for the "or" line between Bearer and key auth.
- There's no vault mapping for this repo yet. The hermes vault write of the plan failed because its command approval timed out (non-interactive), so `Projects/aioz-stream-docs/plans/` is empty.

**2026-09-11 update:** the docs now have an "Audio tracks" section in `import-m3u8.mdx`, matching aioz-stream commit `be9ce236` ("support importing demuxed audio", on `feat/migrate-with-m3u8`).
- A shared group (2 or more referencing variants) imports every rendition.
- A single-variant group must hold exactly 1 rendition.
- Groups no variant references are skipped.
- Either every variant uses an audio group or none does.
- Audio-only masters and media playlists are still rejected.

As of 2026-09-11, `be9ce236` is not in `origin/stag` (5c80c69e) nor in `refactor/sync-with-template-source`. Re-check before publishing.
