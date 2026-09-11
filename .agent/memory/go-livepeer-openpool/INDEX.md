# go-livepeer-openpool — memory index

Facts specific to the `go-livepeer-openpool` fork (`/home/tuan/work/depin-workspace/go-livepeer-openpool`).

- [[fork-delta-lives-on-release-branches]] — `master` is a clean upstream mirror with zero pool code; read the fork via `git diff master origin/release/open-pool/v0.8.10` or `grep "Pool Customization"`.
- [[pool-attribution-architecture]] — what the fork does: ETH-address worker identity over gRPC, S3 event tracker (env-var configured), per-job fee/compute-unit accounting, `/pool/events` feed, webhook worker selection.
- [[codec-support-and-verification-map]] — encoder matrix per accel (Nvidia/Netint are H264+H265 only, no AV1), LPMS not extracted locally, and who verifies transcode output (orchestrator trusts worker-reported pixels; only the gateway re-decodes).
