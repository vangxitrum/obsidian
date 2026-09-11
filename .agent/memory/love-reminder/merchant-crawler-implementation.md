---
type: decision
tags: [crawler, data, architecture, python, go, hcmc]
created: 2026-08-06
agent: main
---

The merchant catalog crawler from [[merchant-catalog-data-plan]] is implemented and verified, scoped to Ho Chi Minh City.

**Split across two services.** The Go backend (`services/backend`) owns the entire schema; `cmd/migrate` is the only thing that runs DDL. The Python crawler (`services/crawler`, uv + Python 3.13) only reads and writes rows. Migrations `000005`-`000009` add geography, merchant/catalog, promotions, crawl/review, and the HCMC seed.

**The crawler never writes canonical tables.** Observations go to `source_records`, proposals to `review_candidates`. Only `ApproveCandidate` in `internal/store/review.go` publishes, and it is deliberately not idempotent: re-approving a decided candidate returns `ErrNotPending` rather than silently reapplying. Operators use `cmd/reviewctl` or `/admin/v1` (constant-time bearer check against `ADMIN_API_TOKEN`, separate from the Clerk-guarded `/v1`).

**Approval order matters.** Merchants before their items and locations, items before their offers. Out-of-order approval returns a 409 naming what to approve first instead of creating an orphan. `source_records.canonical_entity_id` is the provenance link set at approval, and it is how an offer finds its parent item.

**The audit gate is a database CHECK,** `crawl_sources_active_requires_audit`, not application code, so it holds for any client. `crawler audit <domain>` records what robots.txt and the terms mechanically say and files the source as `pending_audit`; a human must read the terms and activate it. An unreadable robots.txt is treated as disallowed, and only a 404 is treated as permissive.

**Why:** Trustworthy freshness and provenance are the product value. A parser bug that could reach published data would destroy it, so the review boundary is absolute rather than advisory.

**How to apply:** Add a new source with `crawler audit`, read its terms, activate, then `crawler run <slug> --max-pages 10` and inspect candidates before approving anything. See [[hcmc-geography-2025-restructuring]] for the address-resolution constraint and [[google-places-review-constraints]] for why Google Maps is not an ingestion path.

On 2026-08-20, `flowersight.com` became the first real active source with
`terms_status = restricted`. Restricted sources strip `description` and
`image_url` before both `source_records` and `review_candidates`; they retain
factual metadata and merchant links. A real capped run exposed and fixed sitemap
index ordering that exhausted the budget on general sitemaps before product
sitemaps. The verified 10-page rerun produced 10 items and 10 offers, all in
stock, priced from 1,450,000 to 5,600,000 VND, with no sale prices detected and
24 total candidates still pending review. The crawler suite passed all 51 tests
against PostgreSQL after these changes.
