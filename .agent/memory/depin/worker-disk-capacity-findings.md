---
type: fact
tags: [depin, worker, capacity, testplanet, orders]
created: 2026-08-07
agent: main
---

Disk-capacity upload tests (TC-FS-14..17) landed 2026-08-07 in
`internal/testplanet/disk_full_test.go`, commit `1142df8` on
`feat/implement-testcases`. Four findings came out of writing them; none are
fixed. Full writeup in `docs/test-plan.md`, "Finding: disk-capacity handling".

**Simulating a full worker:** `cfg.Storage.AllocatedDiskSpace` via
`Reconfigure.Worker`. Exact, because available space never consults the real
filesystem (finding 4).

1. **Order limit is sized to the maximum segment size, not the actual piece.**
   ~34 MiB (35,791,616 B) authorised for a ~32 KiB piece. Admission compares
   free space against that, so a worker needs tens of MB free to accept a piece
   of any size - the tail of every disk is unusable. Same limit-vs-actual gap
   as the upload over-billing fixed on the settlement side; the limit sizing
   itself was never touched.
2. **The order serial is consumed before the space guard runs, so per-piece
   retry is inert.** `verifyOrderLimit` records the serial, then the endpoint
   rejects for space; the client replays the same limit, so attempts 2..N fail
   as replays. Two effects: no retry can ever succeed, for any failure mode;
   and the true cause is erased - a full fleet reports "duplicate or unusable
   order serial" with `not enough space` nowhere in the client's error.
3. **The coordinator never filters selection on capacity** - `disk_space_free`
   is carried and never read, so the worker-side guard is the only defence.
4. **`SharedDisk.DiskSpace` never populates `storageStatus`** - always zero, so
   reported Total/Free are 0, the clamp to real disk size never fires, and
   `MinimumDiskSpace` (500GB default) enforces nothing.

Only production change: the "not enough space" log now records `required`
alongside `available`. That log is what exposed finding 1 - without it a
rejection looks identical to a genuinely full disk.

Related: [[testplanet-test-plan-status]], [[worker-metering-fix]]
