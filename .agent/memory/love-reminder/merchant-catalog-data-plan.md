---
type: decision
tags: [data, merchant, catalog, crawler, promotions, geography]
created: 2026-07-26
agent: main
---

Love Reminder V1 will crawl permission-compatible merchant sources for gifts and experiences in current-boundary Hà Nội, Hồ Chí Minh City, and Đà Nẵng, with narrower service areas modeled explicitly. Purchases and bookings redirect to merchants.

The planned data model separates canonical typed records for merchants, branch locations, contacts, catalog items, offers, and promotions from crawl sources, runs, source records, and review candidates. Crawled changes require review before publication. Promotions can apply merchant-wide, to selected locations, to selected items, or to mixed targets. Raw addresses and source provenance remain preserved.

The finalized two-page ERD and crawl/publication flow are documented in `/home/tuan/personal/tuan/Projects/love-reminder/data-architecture.md`, with the editable source at `Projects/love-reminder/diagrams/merchant-catalog-data-model.drawio`.

**Why:** Trustworthy freshness and provenance are core product value, while canonical records must stay stable and safe from parser errors.

**How to apply:** Implement geography/merchant, catalog/promotion, and crawl/review migrations separately. Treat crawler output as proposed data and apply approved candidates to canonical tables transactionally.
