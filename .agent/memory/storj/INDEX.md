# storj — memory index

Facts specific to the `storj` codebase (`/home/tuan/work/depin-workspace/storj`).

- [[node-self-signed-tags]] — nodes can self-sign tags via `contact.self-signed-tags`; bare `tag:name` attribute lookups ignore the signer, so self-claims can leak into placement.
- [[root-arch-docs-series]] — repo root has a growing series of dense file:line-referenced architecture docs (audit.md, repair.md, reputation.md, reward.md, ...); check `ls *.md` before re-researching a subsystem.
