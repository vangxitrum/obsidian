---
type: decision
tags: [depin, telemetry, monkit, grafana, repair, review]
created: 2026-07-10
agent: claude (final-review-fixes subagent)
---

Branch `feat/coord-metrics-collection`, commit `551e753` "fix(coord/repair,monitoring):
apply final-review polish for metrics branch", on top of `908bdd2`. Applied 3 cheap
polish items from the FINAL whole-branch review (all 11 tasks already individually
approved; "Ready to merge: Yes").

## What changed
1. **repair_strategy consolidation**: `coord/repair/repairer/segments.go`'s
   `repair_strategy_rs`/`repair_strategy_clone` (two separate counters, a deliberate
   Task-4 deviation noted in [[coord-repair-metrics-instrumentation]] and flagged in
   the dashboard as "a possible future refactor") became one `repair_strategy` counter
   tagged `strategy="rs"|"clone"` via `monkit.NewSeriesTag`, matching the
   `order_limits_signed`/`segment_committed` tagging pattern used elsewhere on this
   branch. Updated 3 tests in `segments_test.go` + `monkit_test.go`'s `assertCounter`
   helper (gained optional variadic `tags ...monkit.SeriesTag`, backward compatible
   with existing untagged calls) + the `coord-integrity.json` "Repair Strategy" panel.
2. **order_settlement_bytes dashboard fix**: was `field="ravg"` (average-magnitude
   family); switched to `sum by (action) (rate(order_settlement_bytes{field="sum"}...))`
   to match sibling byte-throughput panels like `segment_committed_bytes`. Also fixed
   title/description/unit (bytes -> Bps) to match.
3. **$role template variable wiring**: all 3 dashboards (`coord-overview.json`,
   `coord-storage-lifecycle.json`, `coord-integrity.json`) defined an identical
   `$role` var (`multi: true`, `includeAll: true`,
   `label_values(process{field="control"}, role)`) that no panel query actually used
   - a dead dropdown. Added `role=~"$role"` (regex matcher, required because
   multi+includeAll means Grafana formats both multi-select and "All" as a
   pipe-alternation regex, not a literal) to every panel target's label selector: 79
   targets total (25+21+33 across the three files). Did this via a small Python
   script (regex-inject into the single `{...}` block each expr has - verified via
   brace-count assertion that no expr had 0 or >1 selector blocks before touching it)
   rather than 79 manual edits; then read the full diff to confirm no
   double-injection and no unrelated panel touched.

## Explicit non-goals (per task scope)
Did NOT rename any `coord/file` metric (`file_created`, `segment_committed`, etc.) -
a human explicitly decided to keep those as shipped despite other review findings.
Did NOT act on any other review finding beyond these 3.

## Verification
`go build ./coord/repair/...` clean; `go test ./coord/repair/... -race -count=1`
all green (4 packages); all 3 dashboard JSONs re-validated with
`python3 -m json.tool`. Report written to
`.superpowers/sdd/final-review-fixes-report.md` in the depin repo (that directory is
gitignored on this branch - existing files there like `task-10-brief.md` and the
`review-*.diff` files are also untracked, so this is consistent with prior practice,
not an oversight).

## Reusable pattern: bulk-editing Grafana dashboard JSON safely
When every panel query across large dashboard JSON files needs the same mechanical
label injected: load with `json.load` (preserves key order in Python 3.7+), regex
each `target["expr"]` for exactly one `{...}` selector block (assert count==1, don't
silently skip mismatches), inject the label before the closing brace, `json.dump`
with `indent=2` + trailing newline to match Grafana's export format. Then
`python3 -m json.tool` each file to confirm still-valid JSON, and grep
`expr-count == injected-label-count` per file as a completeness check before trusting
the diff.
