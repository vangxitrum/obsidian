---
title: BE / API Definition of Done template (Jira checkbox list)
type: project
updated: 2026-08-21
---

# BE / API Definition of Done template

Approved by Tuan on 2026-08-21. Use this verbatim as the DoD for any **backend / API**
subtask. Append it to the existing description - never replace one.

Write it as a real **Jira checkbox list** (ADF `taskList` / `taskItem`, state `TODO`),
not plain bullets, so members tick items off in the ticket.

1. Endpoint implemented to the agreed contract: path, method, request/response schema, status codes
2. All input fields validated; error cases return the documented error shape
3. Unit tests added and passing - happy path + at least one failure/edge case per endpoint
4. Integration tests done, and the full list of test cases posted as a comment on this ticket
5. Lint + format pass, zero new warnings
6. Build and full test suite green in CI
7. Code reviewed and approved by at least 1 reviewer; all comments resolved
8. API doc updated (OpenAPI/Swagger) with an example request and response
9. Deployed to dev/test env and callable by FE (or mock contract published)
10. No remaining blocker; working note updated; Jira status moved to Done

Decisions behind it:
- Item 4 deliberately requires the **test-case list in a comment**, not just a ticked box -
  Tuan's call, because that item is the easiest to tick without doing.
- Research / spike subtasks are **out of scope** for this template (Tuan: "ignore research data").
  AGM-90 was skipped for exactly this reason.
- Applied to Done tickets too, as a record of what should have been checked.

## Subtask title convention

`<ROLE> - <Feature name> - <Action + Object>`, e.g.
`BE - River Gauge Map - Implement River Gauge Stations API`.
Bare `BE - Implement API` is banned - it is the generic-title anti-pattern from
[[jira-task-workflow-standards]] and is indistinguishable on a board.

## How to write it through the Atlassian MCP

`editJiraIssue` with `contentFormat: "adf"` and a description doc of
`heading(level 3) "Definition of Done"` + `taskList` of `taskItem` nodes
(each needs a unique `localId`).

Caveat: the MCP always returns descriptions as markdown, and `expand: renderedFields`
returns a lossy `<ul><li>` HTML render even for a real taskList. The reliable signal that
checkboxes stored correctly is `- [ ]` in the returned markdown (a plain bullet list
serializes as `- ` with no brackets).

Related: [[my-team]], [[jira-site-and-roster]]

## Test-case lists on `BE - Conduct Self-Review & Functional Testing` subtasks

Those subtasks get a **Test Cases** checkbox list in the description (not a DoD), numbered
TC01, TC02, ... and derived from the parent Feature + the sibling `BA - ` subtask. Shape:
contract/schema first, then field-by-field checks, then filters and boundary values, then
error and empty-result paths, then provider failure handling, then one end-to-end render
check naming the FE sibling ticket, and finally "Lint, build and full test suite green in CI".

Applied 2026-08-21 to AGM-97, 98, 99, 101, 102, 103, 104, 105 (all also moved to In Progress).

## AGM workflow transition ids (global, same for every issue)

`11` To Do, `21` In Progress, `31` In Review, `41` Done, `42` Pending.

## Scope cuts decided 2026-08-21 (AGM / Volcano)

- **Ash-related data is not supportable** - no source. Dropped from the volcano API scope:
  ash-plume zone geometry, forecast ash tracks (+6/+12/+18h), ashfall areas (AGM-85),
  and the "Ash forecast (ETA + thickness)" panel row (AGM-93). Scope comments posted on
  both tickets.
- **"Recommended action"** panel row also dropped from the Volcano Detail API (AGM-93).
- Still unreconciled: AGM-28 and AGM-29 (the Feature specs) still list these rows, and
  AGM-84 / AGM-86 / AGM-127 / AGM-128 (FE) still assume they exist. Needs the same cut
  or an alternative source.

## Other gaps found while writing DoDs (2026-08-21)

- **AGM-31** (Tropical Cyclone Detail Panel) has *no description at all*; the spec is only
  inferable from AGM-30. Definition-of-Ready gap - ask anh.ngoc.nguyen to write it.
- **AGM-10** lists "Fire status" with an empty meaning cell - the allowed value set is
  undefined, and AGM-54 + AGM-55 both return it and must agree.
- **AGM-72** ("Validate Weather Module Behavior") holds a full API Definition of Done that
  actually belongs to AGM-71. Copied onto AGM-71; AGM-72 left untouched pending a decision.

## Research-task template (established 2026-08-21)

Data-source research tickets follow one shape: a **feature support matrix**, one row per
feature/field from the parent Feature, with columns Feature | Verdict (Yes/Partial/No) |
Source | Evidence | Limitations | If Partial or No: why + alternative. "Unknown" is banned
as a verdict. Latency/quota figures must be **measured**, never copied from vendor docs.
Findings get posted as a comment on the ticket. See AGM-89 for the original.

Tickets using it: AGM-89 (airport), AGM-140 (earthquake EW), AGM-141 (tsunami),
AGM-142 (sea ice Arctic), AGM-143 (sea ice Antarctic), AGM-144 (ENSO).

## New work created 2026-08-21

- **AGM-139** Feature "Earthquake Early Warning" under Epic AGM-1, scope deliberately
  undefined pending AGM-140's findings. BA writes requirements after the research.
- Research owners as assigned by Tuan: earthquake + tsunami -> nam.quoc.le,
  sea ice (both hemispheres) -> Anh Dang, ENSO -> khoi.quang.le.
- Existing structure found (do NOT re-create): Epic AGM-109 ENSO -> AGM-117/118/119
  (map layer, detail panel, time series); Epic AGM-111 Sea ice extent -> AGM-114/115/116
  (same three shapes); Epic AGM-1 -> AGM-2, AGM-3, AGM-4.

## Reverting a Done transition (verified 2026-08-24)

Transitioning back out of Done via `transitionJiraIssue` (id 11 To Do / 21 In Progress)
clears the resolution automatically on this site - no explicit `{"resolution": null}` edit
was needed. Verified across 9 issues.

Scope note: **hoang.huy.nguyen is NOT on Tuan's team** and owns most FE self-review
subtasks in AGM. Bulk status changes scoped to "my team" must exclude him.

## Checkbox gotcha by project type (verified 2026-08-24)

Passing `- [ ]` via `contentFormat: "markdown"` does NOT become a checkbox in **classic**
projects (`simplified: false` - AST, AP, ALP). It comes back escaped as `\[ \]` inside a
plain bullet list. Next-gen projects (`simplified: true` - AGM, AS, DP, FETEAM) convert it.

Always send an explicit ADF `taskList`/`taskItem` for a DoD regardless of project type.
Confirm it worked by re-reading the description: real task items serialize back as `- [ ]`,
a plain bullet list serializes as `- ` with no brackets.

## Wind data source decision (2026-09-03)

AGM-164: the wind data source is **Open-Meteo (JSON API), not GRIB** - despite the ticket
originally being titled "BE - Wind grib API". The **existing source logic must be kept**,
not deleted; Open-Meteo is an added, config-selectable path.

Key design consequence: Open-Meteo answers **per coordinate**, not as a gridded file, so a
wind *field* layer needs a sampling grid + batching + server-side caching, and it returns
**speed/direction**, not U/V components. The old source name is still not recorded anywhere.
