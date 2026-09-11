---
type: decision
tags: [workers, liveness, coordinator, database]
created: 2026-08-26
agent: main
---

Worker liveness is derived from a successful coordinator ping-back recorded in `last_seen`, using `contact.DefaultLastSeenWindow` (4 hours). The old persisted `workers.is_online` boolean was removed because it only reflected the most recent check-in and could remain true indefinitely after a worker disappeared.

**How to apply:** use `last_seen >= now - DefaultLastSeenWindow` for worker status, placement/repair eligibility, and admin reporting. Migration `000052_remove_worker_is_online` drops the obsolete database column.
