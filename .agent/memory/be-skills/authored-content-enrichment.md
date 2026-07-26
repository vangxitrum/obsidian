---
type: decision
tags: [curriculum, content, architecture]
created: 2026-07-22
agent: main
---

All 75 topics now include authored, topic-specific production content: one concrete scenario, three failure modes, four ship/review checks, and two authoritative primary references. This adds 75 scenarios, 225 failure modes, 300 checks, and 150 references.

Content lives in `data/enrichment1.ts` through `data/enrichment5.ts`, aligned with the five source curriculum files. `lib/data.ts` merges enrichment into every `SkillWithMeta` and throws at module initialization if a source skill has no enrichment.

**Why:** Derived metadata improved navigation but did not deepen the curriculum itself. Separate enrichment files keep long-form production guidance maintainable without making the original competency files harder to edit.

**How to apply:** Every newly added skill must receive a matching `ContentEnrichment` entry. Prefer concrete operational scenarios, cause-and-consequence failure descriptions, executable checks, and official specifications or documentation.
