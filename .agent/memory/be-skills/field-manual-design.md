---
type: decision
tags: [design, curriculum, frontend, data]
created: 2026-07-22
agent: main
---

The user selected the Field Manual visual direction and all-skills structured enrichment for the backend engineering curriculum. The interface uses warm paper surfaces, dark green ink, coral annotations, numbered field notes, curriculum metrics, prerequisites, generated outcomes, and related-topic navigation.

Structured metadata is derived centrally in `lib/data.ts` from each topic's existing competency, practice, and interview data so all 75 topics remain consistent without duplicating metadata in source files.

**Why:** The original interface had deep content but behaved like a long document. The redesign exposes learning sequence, scope, outcomes, and practice affordances before users begin reading.

**How to apply:** Preserve the Field Manual language and derive cross-cutting learning metadata centrally when adding topics. Add bespoke source fields only when information cannot be inferred reliably.

The curriculum sidebar is intentionally focused rather than fully expanded: only the current module opens by default, search and level filters remove nonmatches, navigation opens and scrolls to the active topic, module headers show assessed/total counts, and a fixed footer shows overall manual position. Preserve this behavior as the curriculum grows.
