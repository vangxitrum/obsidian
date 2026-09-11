---
type: fact
tags: [depin, download, storj, eestream, grpc, flow-control, throughput]
created: 2026-08-19
agent: main
---

# Download stats vs Storj source

The 2026-08-19 `download-stats.md` evidence proves downstream backpressure, but
does not prove its conclusion that large downloads execute 16 global
`maxStripesAhead` barrier rounds.

- AIOZ and Storj both use `maxStripesAhead = 256`, `ShareSize = 256`, a 32 KiB
  eestream batch/output buffer, and a K-of-N watermark decoder. The decoder can
  use any K readers at the next watermark; it does not wait for the same fixed K
  every round. Storj's slow-reader tests explicitly prove surplus slow readers
  do not gate reconstruction.
- For RS(29,...) a 32 KiB decoded output buffer holds only 4 whole 7,424-byte
  stripes. The decoder advances in batches of at most 4 stripes, while piece
  readers may remain 256 stripes ahead. It does not consume one 256-stripe
  global round at a time.
- `StreamingPiece.ReadSharesFrom` reads a 32 KiB batch before checking the
  strict `> maxStripesAhead` condition. With 256-byte shares, a fast reader can
  reach 384 shares / 96 KiB ahead, not a strict 64 KiB. The AIOZ response reader
  can additionally retain half of a 64 KiB response frame.
- AIOZ's 64 KiB worker framing alone explains multiple sends per piece. A 30 MiB
  plaintext object expands to 32 MiB under 240->256 AES-GCM blocks, then pads to
  4,520 RS stripes: each piece is 1,157,120 bytes (1.1035 MiB), or 18 x 64 KiB
  response chunks including the tail. `download-stats.md` incorrectly divides
  plaintext by K and reports 1.03 MiB / about 16 chunks.
- `piece_slowest ~= request_total` is largely tautological. Every lazy reader
  records its lifetime in `Close`, and decoder close closes unfinished candidates
  at segment completion, so a loser naturally spans approximately the segment
  wall time. It is not independent proof that one fixed piece gated the request.
- AIOZ's coordinator uses Storj's `DownloadNodes` formula. RS(29,52,60,80)
  yields 38 signed candidates. AIOZ's SDK activates only K+6=35 and parks 3;
  Storj starts every candidate selected by its satellite. Activating all 38 may
  improve tail choice but adds traffic on AIOZ's shared path, so benchmark it or
  prefer delayed/adaptive hedging rather than blindly increasing fanout.

The strongest source-supported untested suspect is grpc-go flow control:

- AIOZ's worker response frame is 64 KiB.
- grpc-go v1.74.2 defaults the per-RPC HTTP/2 window to 65,535 bytes and the
  write quota to 64 KiB.
- AIOZ sets neither `grpc.WithInitialWindowSize` nor
  `grpc.WithInitialConnWindowSize` on its client. Flow-control window updates
  are driven by application reads. This exactly matches worker `Send` blocking
  and the edge's roughly corresponding inter-`Recv` gap.
- Storj uses DRPC over direct hybrid TCP/QUIC, requests 16 KiB piece messages,
  initially authorizes 256 KiB, and therefore does not have AIOZ's HTTP/2 64 KiB
  application-read window in this path.

Highest-value causal experiment: sweep only the AIOZ gRPC client's inbound
stream/connection windows (64 KiB, 256 KiB, 1 MiB) while keeping eestream at 256,
then separately sweep `maxStripesAhead` (256 vs 2048). Compare worker Send,
edge inter-Recv gap, and end-to-end throughput with enough repeated runs. Do not
change both controls together.

Storj can still suffer the inherent Kth-fastest-reader limit and overfetch. It is
less exposed to AIOZ's observed form because its piece connections directly fan
out over independently operated node links instead of AIOZ's nearly all-relayed
shared topology, and because its transport lacks the gRPC 64 KiB coupling. Source
alone cannot establish Storj production throughput, so "Storj does not hit it"
must remain qualified rather than asserted.

Related: [[download-phase-tracing]], [[download-network-cap-root-cause]],
[[gosdk-streaming-reconstruction]], [[worker-holepunch-never-upgrades]].
