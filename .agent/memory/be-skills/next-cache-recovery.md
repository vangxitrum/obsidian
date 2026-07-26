---
type: reference
tags: [nextjs, build, debugging]
created: 2026-07-22
agent: main
---

A production build writing to `.next` while `next dev` was running replaced the development React Client Manifest. The live server then returned 500 for every route with `Could not find the module ... in the React Client Manifest`; the public tunnel returned 502 because requests stalled. An earlier manifestation also reported a missing Supabase vendor chunk.

Recovery: stop the dev server, move the mixed `.next` directory out of the workspace, and restart development. `next.config.ts` now uses `.next-dev` for development and `.next` for production, preventing builds from corrupting the live server. This was verified by running a full production build while development remained live; local and public `/skill/rest` and `/skill/git-advanced` continued returning HTTP 200.
