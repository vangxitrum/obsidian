# storj — memory index

Facts specific to the `storj` codebase (`/home/tuan/work/depin-workspace/storj`).

- [[node-self-signed-tags]] — nodes can self-sign tags via `contact.self-signed-tags`; bare `tag:name` attribute lookups ignore the signer, so self-claims can leak into placement.
