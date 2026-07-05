---
type: note
tags: [architecture, performance]
created: 2026-07-05
---

# Caching layer idea

Add a read-through cache in front of the profile service to cut repeated DB hits.
Worth prototyping against the [[2026-07-05]] load-test numbers.
