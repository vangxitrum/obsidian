---
type: fact
tags: [go-livepeer-openpool, livepeer, fork, orientation]
created: 2026-08-25
agent: main
---

`/home/tuan/work/depin-workspace/go-livepeer-openpool` is the
`Livepeer-Open-Pool/go-livepeer-openpool` fork of `livepeer/go-livepeer`
(module path is still `github.com/livepeer/go-livepeer`, README is stock upstream).

**Non-obvious: `master` carries ZERO pool code.** It is a clean mirror of upstream
(currently v0.8.10). Reading `master` tells you nothing about what the fork does.
The pool feature set lives only on release branches:

- `origin/release/open-pool/v0.8.10` - single squashed commit "Initial Commit - AI
  Enabled Pools", ~2100 insertions over 20 files. **This is the current one.**
- `origin/release/open-pool/v0.8.8` - previous release.
- `origin/release/ai-pool-enablement` - based on much older upstream (diff vs master
  shows ~42k deletions); stale, do not use as the reference.

So: `git diff master origin/release/open-pool/v0.8.10` is the way to read the fork.
Every pool hunk is marked with a `// ** Pool Customization **` comment, so
`grep -rn "Pool Customization"` on a checked-out release branch enumerates the delta.

See [[pool-attribution-architecture]] for what the delta actually does.
