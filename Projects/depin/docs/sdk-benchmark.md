---
type: doc
project: depin
tags: [depin, uplink-sdk, benchmark, testing]
created: 2026-07-05
---

# sdk-benchmark

Networked benchmark for the uplink SDK. Exercises the full client path — register account -> create contract -> upload -> download -> integrity-check — over the wire against a live coordinator and worker(s). It is the networked counterpart to `pipeline-benchmark` (which isolates the CPU engine with no network).

Source: `uplink-sdk/cmd/sdk-benchmark/main.go`

## What it does

1. Builds an SDK client from an identity directory.
2. (Optional) Registers the client account on the coordinator.
3. Picks a placement and creates a storage contract.
4. Uploads N files of deterministic seeded random data.
5. Downloads M files back and SHA-256 verifies each against its seed.
6. Prints a Go-benchmark-style timing table plus a per-stage breakdown.

Data is a deterministic seeded random stream (O(1) memory, any size), so no test files are needed on disk. A non-zero exit code means an integrity mismatch.

## Prerequisites

- A running coordinator and at least one worker (a dev cluster).
- An identity directory containing:
  - identity.cert, identity.key (always required)
  - priv_key.json (required only when using -register)
  - piece_key.json (optional)
- The coordinator identity UUID and its host:port.

## Quick start

    go run -tags purego ./uplink-sdk/cmd/sdk-benchmark \
      -identity-dir ./identities/uplink \
      -host <host:port> -coord-id <coord-uuid> \
      -email you@aioz.io \
      -size 1MiB -uploads 3 -downloads 5 -workers 4

## Flags

### Connection
- -identity-dir  Path to the identity directory (required).- -coord-url     Full coord PeerURL "<coord-id>@host:port". Overrides -host/-coord-id.
- -host          Coord remote host as "host:port" (used with -coord-id when -coord-url is unset).
- -coord-id      Coord identity UUID (used with -host when -coord-url is unset).

You must provide either -coord-url, OR both -host and -coord-id.
### Registration

- -register  Register the client account on coord before benchmarking (default true). Needs priv_key.json in the identity dir.
- -email     Account email for registration (required when -register is set).

Registration is not idempotent on the coordinator. If the account already exists, the tool falls back to GetAccount and logs "client already registered", so re-runs are safe. Use -register=false to skip the step entirely for an already-registered identity.

### Workload
- -size        Bytes per file, e.g. 64MiB (default 64MiB).
- -uploads     Number of files to upload (0 = fall back to -count / profile).
- -downloads   Number of downloads (0 = same as uploads). Round-robins over the uploaded set, so it may exceed or be fewer than -uploads.
- -count       Legacy: seeds both -uploads and -downloads when neither is set (default 5).
- -profile     Size/count preset: small | medium | big. Explicit -size/-count override it.
- -workers     Concurrent upload/download workers (default 1).
- -download    Download each uploaded file back (default true).
- -verify      SHA-256 hash-check each download against the seed (default true).

### Profiles
- small   64KiB x 1000 files
- medium  64MiB x 10 files
- big     1GiB x 1 file

### Run control / profiling
- -region      Contract region (default "global").
- -placement-id  Placement id to use; -1 picks the first available.
- -timeout     Overall deadline for the whole run (default 10m).
- -quiet       Suppress per-op SDK logs; only print the summary.
- -benchstat   Emit raw Go-benchmark lines (for benchstat) instead of the table.
- -cpuprofile  Write a CPU profile to this path.
- -memprofile  Write a memory profile to this path.

## Examples

Independent upload/download counts:

    go run -tags purego ./uplink-sdk/cmd/sdk-benchmark \
      -identity-dir ./identities/uplink -host <host:port> -coord-id <uuid> \
      -email you@aioz.io -size 1MiB -uploads 10 -downloads 50 -workers 8

Skip registration (identity already registered):

    go run -tags purego ./uplink-sdk/cmd/sdk-benchmark \
      -identity-dir ./identities/uplink -coord-url "<uuid>@host:port" \
      -register=false -profile medium

Big-file throughput with a CPU profile:

    go run -tags purego ./uplink-sdk/cmd/sdk-benchmark \
      -identity-dir ./identities/uplink -host <host:port> -coord-id <uuid> \
      -email you@aioz.io -profile big -cpuprofile cpu.out

## Reading the output

- The summary table shows Upload and Download phases with Ops, Bytes, Wall, and Avg/Min/Max latency.
- The stage breakdown shows where each transfer spent its time (share of summed stage time).
- integrity: X/Y ok reports how many downloads passed the hash check. A mismatch fails the run with a non-zero exit.

## Notes

- Downloads round-robin over uploaded files, so with -uploads 3 -downloads 5 the tool re-downloads some files; all must still pass the integrity check.
- The account UUID is derived from the identity's mTLS certificate, not the -email; the email is metadata on the account record.
