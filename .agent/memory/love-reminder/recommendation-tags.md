---
type: decision
tags: [recommendations, taxonomy, crawler, review]
created: 2026-08-22
agent: main
---

Recommendation tags are a controlled, Vietnamese-first taxonomy stored in `recommendation_tags` and `recommendation_tag_aliases`, with confirmed many-to-many assignments on merchants and catalog items. Dimensions are category, occasion, recipient, interest, and style. Price, sale, stock, city, and lead time remain structured fields rather than tags.

Crawler tag suggestions are deterministic advisory metadata under `proposed_data.tag_suggestions`; each suggestion carries matching evidence. Suggestions are deliberately excluded from content diffs and never publish automatically. The review console preselects suggestions but requires a human approval action. An omitted `selectedTagSlugs` decision preserves existing assignments, while an explicitly empty list clears them.

`crawler retag <source-slug>` refreshes suggestions on existing pending merchant/item proposals without creating fake source changes or defeating conditional HTTP requests. This is also the mechanism to use after taxonomy or alias updates.

**Why:** Recommendation filters need stable semantics and provenance, while crawler text matching is not trustworthy enough to become canonical without review.

**How to apply:** Add or retire taxonomy entries in migrations, enrich aliases conservatively, run `crawler retag <source-slug>`, then confirm suggestions in `/review/[id]`. Related: [[merchant-crawler-implementation]], [[review-console]].
