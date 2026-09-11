# aioz-stream-core — memory index

- [[gosdk-system-headers-upload]] — wired Content-Type + Cache-Control as go-sdk "system headers" into every StorageHelper upload call.
- [[highbitdepth-sdr-profile-crash]] — fixed 10-bit-but-SDR sources crashing on `-profile:v main` (the existing HDR fix only caught PQ/HLG, not plain bt709 10-bit).
- [[merge-new-cdn-update-transcode]] — fixed the c0a3847 merge: Upload contractID build break, dropped -initial_offset A/V sync, deleted superseded probe.go and stale tests.
- [[gosdk-contract-id-required]] — empty contract_id makes go-sdk mint a phantom contract; coordinator answers NotFound "file: not found" after the whole transcode. Guarded with ErrMissingContractID.
- [[debugging-worker-jobs]] — where to look when a job fails: job-db queries, docker log rotation, which fields prove how far a playlist got.
- [[hls-upload-master-read]] — HLS uploader opened a per-playlist master.m3u8 ffmpeg no longer writes (since 9e7ce9a), failing every HLS rendition; fix + e2e guard on fix/hls-upload-master-read.
