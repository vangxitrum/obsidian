---
title: Merchant Catalog Crawler - Ho Chi Minh City First
date: 2026-08-06
tags:
  - project/love-reminder
  - plan
  - crawler
  - data
status: implemented
---

# Merchant Catalog Crawler - Ho Chi Minh City First

## Context

Phase 1 of Love Reminder shipped occasions, reminders, and push delivery, but the README explicitly states "Merchant crawling is planned separately and is not part of Phase 1." The design work was completed on 2026-07-26 and lives as a two-page draw.io diagram plus a summary note:

- `/home/tuan/personal/tuan/Projects/love-reminder/data-architecture.md`
- `/home/tuan/personal/tuan/Projects/love-reminder/diagrams/merchant-catalog-data-model.drawio` (page 1: ERD, page 2: crawl and publication flow)

Nothing from that design exists in code yet. The current schema stops at `000004_auth_identities`, and there is no crawler process.

This plan implements that design end to end, scoped to Ho Chi Minh City so the pipeline is proven against one real city before Ha Noi and Da Nang are added. The intended outcome: a running crawler that fetches permission-compatible HCMC merchant sources, stages every observation with provenance, proposes changes as review candidates, and applies approved candidates transactionally into canonical typed tables that the product can later read from.

The core product guarantee driving the design is that canonical data stays trustworthy: parser output is never published directly, and a stale price is never presented as current.

### Decisions taken

| Question | Decision |
|---|---|
| Crawler placement | Separate Python service at `services/crawler` with its own `pyproject.toml` and `Dockerfile`, writing to the same PostgreSQL |
| Scope | Full pipeline: migrations, HCMC geography seed, crawler, staging, review, approval, admin API. No mobile recommendation feed |
| Sources | I research and audit candidate HCMC merchant domains during implementation; only sources that pass robots and terms checks get seeded active |
| Review surface | Admin HTTP API behind a token plus a `reviewctl` Go CLI. No web UI |

### Ownership boundary

The Go backend owns the schema. `services/backend/cmd/migrate` remains the only thing that runs DDL. The Python crawler only reads and writes rows. This keeps one migration history and avoids two services racing on schema.

---

## 1. Schema migrations

All new files go in `services/backend/migrations/`, continuing the existing numbering and following the conventions in `000001_initial.up.sql`: `uuid PRIMARY KEY DEFAULT gen_random_uuid()`, `timestamptz NOT NULL DEFAULT now()`, `CHECK (x IN (...))` for enums, and a `set_updated_at` trigger per table. The `set_updated_at()` function already exists from `000001` and must be reused, not redefined.

Every `.up.sql` needs a matching `.down.sql` that drops in reverse dependency order.

### `000005_geography.up.sql`

- `cities` - `code` unique, constrained to `('HN','HCM','DN')`, plus `name`, `slug`, `active`
- `administrative_areas` - `city_id` FK, self-referencing `parent_id`, `official_code`, `name`, `area_type` in `('ward','commune','district','town')`, `active`. `UNIQUE (city_id, official_code)`
- `administrative_area_aliases` - `area_id` FK, `alias`, `normalized_alias`, `source` in `('legacy','colloquial','crawled')`. Index on `normalized_alias` for address resolution lookups

`normalized_alias` is the lowercase, diacritic-stripped, whitespace-collapsed form. Normalization happens in Python at write time so Postgres does not need `unaccent`.

### `000006_merchant_catalog.up.sql`

- `merchants` - `slug` and `canonical_domain` both unique, `name`, `website_url`, `merchant_type`, `verification_status` in `('unverified','auto_verified','human_verified')`, `status` in `('active','paused','delisted')`, `first_seen_at`, `last_verified_at`
- `merchant_locations` - `merchant_id`, `city_id`, `administrative_area_id` (nullable), `raw_address` (never overwritten), `normalized_address`, `lat`/`lng` as `numeric(9,6)`, `opening_hours jsonb`, `google_place_id text` (nullable, unique where not null), `status`
- `merchant_contacts` - `merchant_id`, nullable `location_id`, `contact_type` in `('phone','email','messenger','zalo','other')`, `raw_value`, `normalized_value`, `is_primary`, `verified_at`
- `catalog_items` - `merchant_id`, `item_type` in `('gift','experience')`, `title`, `category`, `image_url`, `status`, `first_seen_at`, `last_verified_at`
- `catalog_item_availability` - `item_id`, `city_id`, nullable `location_id`, `fulfillment_type` in `('pickup','delivery','onsite','shipping')`, `lead_time_hours`, `active`
- `catalog_offers` - `item_id`, `source_id` FK to `crawl_sources`, `merchant_url`, `regular_price_vnd bigint`, `sale_price_vnd bigint`, `availability` in `('in_stock','out_of_stock','preorder','unknown')`, `valid_from`, `valid_until`, `last_seen_at`

Prices are `bigint` VND minor-unit-free integers. VND has no subunit in practice, so store whole dong.

The `catalog_offers.source_id` FK is why `000006` must run after the crawl tables, or the FK must be added in `000008`. Simplest ordering: create `crawl_sources` in `000006` ahead of `catalog_offers`, or defer the FK. Add the FK in `000008` with `ALTER TABLE` to keep each migration thematically clean.

### `000007_promotions.up.sql`

- `promotions` - `merchant_id`, `title`, `terms`, `promotion_code`, `discount_type` in `('percent','amount','gift','other')`, `discount_value numeric`, `scope` in `('merchant','location','item','mixed')`, `starts_at`, `ends_at`, `status`, `landing_url`
- `promotion_locations` - composite PK `(promotion_id, location_id)`
- `promotion_items` - composite PK `(promotion_id, item_id)`

Add a `CHECK` that `ends_at IS NULL OR ends_at > starts_at`, and a partial index on `(merchant_id)` where `status = 'active'`.

### `000008_crawl_review.up.sql`

- `crawl_sources` - nullable `merchant_id`, `base_url`, `adapter_key`, `source_type` in `('website','feed','api')`, `robots_allowed boolean` (nullable = unaudited), `terms_status` in `('allowed','restricted','prohibited','unknown')`, `status` in `('pending_audit','active','paused','blocked')`, `crawl_interval_minutes`, `next_crawl_at`, `last_crawled_at`, `max_requests_per_minute`. `UNIQUE (base_url, adapter_key)`
- `crawl_runs` - `source_id`, `started_at`, `finished_at`, `status` in `('running','succeeded','failed','cancelled')`, and metric columns `pages_fetched`, `records_extracted`, `changes_detected`, `errors`, `error_message`
- `source_records` - `source_id`, `run_id`, `record_type` in `('merchant','location','contact','item','offer','promotion')`, `external_key`, `source_url`, `content_hash`, `extracted_data jsonb`, `http_etag`, `http_last_modified`, `first_seen_at`, `last_seen_at`. `UNIQUE (source_id, record_type, external_key)`
- `review_candidates` - `source_record_id`, `entity_type`, `target_entity_id uuid` (null for new entities), `proposed_data jsonb`, `difference jsonb`, `confidence numeric(3,2)`, `status` in `('pending','approved','rejected','superseded')`, `reviewed_by text`, `reviewed_at`. Partial index on `(status)` where `status = 'pending'`

Also in this migration: `ALTER TABLE catalog_offers ADD CONSTRAINT ... FOREIGN KEY (source_id) REFERENCES crawl_sources(id)`.

`source_records` carries the conditional-HTTP validators so a re-crawl can send `If-None-Match` and `If-Modified-Since` and skip unchanged pages entirely.

### `000009_hcm_geography_seed.up.sql`

Seeds the `HCM` city row and its administrative areas.

**Important and must be verified during implementation, not assumed:** Vietnam restructured local government effective 2025-07-01. Ho Chi Minh City absorbed Binh Duong and Ba Ria - Vung Tau, and the district tier was abolished in favour of a two-tier city to ward/commune structure. The memory note `merchant-catalog-data-plan` records the decision to "keep current city boundaries", so the seed must use the current post-restructuring ward list, not the legacy 22-district list.

First implementation step for this migration: fetch the current official ward and commune list for HCMC from an authoritative source (General Statistics Office or the government administrative-unit dataset) and generate the seed from it. Do not hand-type from memory.

Legacy district names (`Quận 1`, `Quận 3`, `Bình Thạnh`, `Thủ Đức`, `Gò Vấp`, and so on) go into `administrative_area_aliases` with `source = 'legacy'`, mapped to the wards that replaced them. Crawled merchant addresses will use old names for years, so alias resolution is what makes address matching work at all. Where a legacy district maps to many wards, insert one alias row per ward and let the resolver treat a multi-hit alias as district-level context rather than a precise area match.

---

## 2. Python crawler service

New directory `services/crawler/`. Managed with `uv` (not currently installed - add to `mise.toml`).

```
services/crawler/
  pyproject.toml
  uv.lock
  Dockerfile
  README.md
  .env.example
  src/crawler/
    __init__.py
    config.py          # env loading, mirrors services/backend/internal/config
    db.py              # asyncpg pool, transaction helpers
    models.py          # pydantic models for extracted records
    robots.py          # protego-based robots.txt fetch + cache + crawl-delay
    fetch.py           # httpx AsyncClient, conditional GET, per-host limiter
    normalize/
      text.py          # diacritic strip, slugify, title normalization
      address.py       # VN address parse + area resolution via aliases
      phone.py         # phonenumbers -> +84 E.164
      price.py         # "1.250.000đ" / "1,250,000 VND" -> 1250000
    adapters/
      base.py          # Adapter protocol
      registry.py      # adapter_key -> class
      jsonld.py        # schema.org Product/Offer/LocalBusiness via extruct
      sitemap.py       # sitemap.xml + robots Sitemap: discovery
    dedupe.py          # content_hash, external_key derivation
    diff.py            # proposed vs published -> difference jsonb + confidence
    pipeline.py        # the run loop for one source
    scheduler.py       # poll due sources, dispatch, per-host concurrency
    audit.py           # robots + terms audit for a candidate domain
    cli.py             # typer entrypoints
  tests/
    fixtures/hcm/...   # recorded HTML/XML from real HCMC merchant pages
    test_*.py
```

Dependencies: `httpx`, `asyncpg`, `pydantic`, `extruct`, `selectolax`, `protego`, `phonenumbers`, `python-slugify`, `tenacity`, `structlog`, `typer`. Dev: `pytest`, `pytest-asyncio`, `respx`, `ruff`, `mypy`.

### Scheduling

No River equivalent in Python and none is needed. `scheduler.py` runs an asyncio loop that claims due sources with

```sql
SELECT * FROM crawl_sources
WHERE status = 'active' AND next_crawl_at <= now()
ORDER BY next_crawl_at
FOR UPDATE SKIP LOCKED LIMIT $1
```

inside a transaction, immediately pushing `next_crawl_at` forward so a crash cannot cause a hot loop. `FOR UPDATE SKIP LOCKED` gives safe multi-replica operation with no extra infrastructure. Per-host concurrency is an `asyncio.Semaphore` keyed by host, sized from `max_requests_per_minute`.

### Fetch politeness

Non-negotiable behaviours, matching the flow diagram's "Fetch" node:

- A descriptive `User-Agent` that identifies the crawler and gives a contact URL
- `robots.txt` checked per URL via `protego`, honouring `Crawl-delay`
- `Retry-After` respected on 429 and 503, with `tenacity` exponential backoff otherwise
- Conditional requests using stored `http_etag` / `http_last_modified`; a `304` short-circuits parsing and only bumps `last_seen_at`
- Hard cap on pages per run, recorded in `crawl_runs.pages_fetched`

### Pipeline

`pipeline.run_source(source)` performs: open `crawl_run` -> discover URLs via adapter -> fetch -> parse to typed `models.py` records -> normalize (address, phone, price, title) -> compute `content_hash` and `external_key` -> upsert `source_records` -> for each record whose hash changed or which has no published counterpart, compute the diff against canonical tables and insert a `review_candidate` -> mark any older `pending` candidate for the same `(entity_type, target_entity_id)` as `superseded` -> close `crawl_run` with metrics.

Unchanged records only touch `last_seen_at`. That is what makes daily offer re-checks cheap.

Confidence scoring lives in `diff.py` and is a simple weighted sum: exact domain match, SKU or external key stability, city and area resolution certainty, and completeness of required fields. Low-confidence candidates still get created; they just sort last in review.

---

## 3. Source audit for HCMC

Before any crawling, `crawler audit <domain>` performs and records:

1. Fetch and parse `robots.txt`; record `robots_allowed` for the intended paths
2. Fetch the terms of service page and flag it for human reading; record `terms_status`
3. Detect a machine-readable surface (JSON-LD, sitemap, product feed, public API) and pick an `adapter_key`
4. Insert or update the `crawl_sources` row with `status = 'pending_audit'`

A source only moves to `status = 'active'` after a human confirms terms permit automated access. Candidate categories for HCMC gifts and experiences: florists and gift shops, spa and wellness booking, restaurant and dining experiences, workshop and class providers, cake and confectionery. Audit results get written to `services/crawler/docs/hcm-source-audit.md` with date, robots verdict, terms verdict, and decision, so the provenance of the decision itself is recorded.

Google Maps scraping is not an acceptable ingestion path. Per the `google-places-review-constraints` memory note, Google Places enrichment is a separate, later, API-based concern; this plan only reserves the `google_place_id` column on `merchant_locations` for it.

---

## 4. Go backend: domain, store, review

### New files

- `services/backend/internal/domain/catalog.go` - `Merchant`, `MerchantLocation`, `CatalogItem`, `CatalogOffer`, `Promotion`
- `services/backend/internal/domain/crawl.go` - `CrawlSource`, `CrawlRun`, `SourceRecord`, `ReviewCandidate`
- `services/backend/internal/store/catalog.go` - canonical reads and writes
- `services/backend/internal/store/review.go` - candidate listing and the approval transaction

Follow the existing style in `internal/store/store.go`: methods on `*Store`, raw SQL in backticks, `fmt.Errorf("verb noun: %w", err)` wrapping, `mapNotFound` for `pgx.ErrNoRows`, and `s.Pool.Begin` with `defer tx.Rollback(ctx)` for transactions. `EnsureIdentityProfile` (store.go:35) is the closest existing model for a multi-statement transactional write.

### `ApproveCandidate`

The heart of the design. One transaction:

1. `SELECT ... FOR UPDATE` the candidate; reject unless `status = 'pending'`
2. Switch on `entity_type` and upsert the canonical row from `proposed_data`, inserting when `target_entity_id` is null and updating otherwise
3. Set `last_verified_at = now()` on the touched canonical row
4. Mark the candidate `approved` with `reviewed_by` and `reviewed_at`
5. Mark sibling pending candidates for the same target `superseded`
6. Commit

`RejectCandidate` is the same shape without step 2 or 3. Canonical data is never touched by the crawler process itself, only by this function.

### Admin API

Add to `config.Config` (`internal/config/config.go`) an `AdminAPIToken` read from `ADMIN_API_TOKEN`, required whenever `AuthDisabled` is false. Add a small admin middleware using `crypto/subtle.ConstantTimeCompare` on a bearer token, mounted as a separate `router.Route("/admin/v1", ...)` group in `internal/httpapi/server.go` alongside the existing `/v1` group. It must not use `authMiddleware` or `ensureProfile`, since admins are not app profiles.

```
GET    /admin/v1/review-candidates?status=pending&city=HCM&entity_type=&limit=&cursor=
GET    /admin/v1/review-candidates/{id}
POST   /admin/v1/review-candidates/{id}/approve
POST   /admin/v1/review-candidates/{id}/reject
GET    /admin/v1/crawl-sources
PATCH  /admin/v1/crawl-sources/{id}      # status, interval, terms_status
GET    /admin/v1/crawl-runs?source_id=
```

Handlers reuse the existing `writeJSON`, `writeError`, `decode`, `respond`, and `pathUUID` helpers in `server.go`.

### `cmd/reviewctl`

Thin Go CLI over the same store package (direct DB, not HTTP) for bulk work:

```
go run ./cmd/reviewctl list --city HCM --status pending
go run ./cmd/reviewctl show <id>
go run ./cmd/reviewctl approve <id> [--by tuan]
go run ./cmd/reviewctl reject  <id> --reason "wrong price parse"
go run ./cmd/reviewctl approve-all --source <slug> --min-confidence 0.9
```

---

## 5. Wiring and docs

- `mise.toml` - add `python = "3.13"` and `uv = "latest"` next to the existing `go` and `node` pins
- `compose.yaml` - add a `crawler` service building `services/crawler`, depending on `postgres` healthy, with `DATABASE_URL` pointed at the compose network. Keep it `profiles: [crawler]` so `docker compose up -d postgres` stays the default lightweight path documented in the README
- `services/backend/.env.example` - add `ADMIN_API_TOKEN=`
- `services/crawler/.env.example` - `DATABASE_URL`, `CRAWLER_USER_AGENT`, `CRAWLER_CONTACT_URL`, `CRAWLER_MAX_CONCURRENCY`, `CRAWLER_DEFAULT_INTERVAL_MINUTES`
- `services/backend/README.md` - document the admin endpoints and `reviewctl`
- Root `README.md` - replace "Merchant crawling is planned separately and is not part of Phase 1" with the crawler quickstart

---

## Verification

Per the project rule preferring end-to-end tests over unit tests for product behaviour, the primary gate is a full-pipeline test, not a pile of mocks.

**Schema**

```bash
docker compose up -d postgres
cd services/backend
export DATABASE_URL='postgres://love_reminder:love_reminder@localhost:5441/love_reminder?sslmode=disable'
go run ./cmd/migrate
```

Then verify the down path by running `migrate down` to `000004` and back up, confirming no orphaned objects.

**Go**

```bash
cd services/backend
go vet ./...
go test ./...            # with DATABASE_URL set so the integration tests do not skip
docker build .
```

New integration tests in `internal/store/` follow `store_integration_test.go`: skip when `DATABASE_URL` is unset, clean up their own rows via `t.Cleanup`. Cover `ApproveCandidate` creating a new merchant, `ApproveCandidate` updating an existing offer price, and supersession of sibling candidates.

**Python**

```bash
cd services/crawler
uv sync
uv run ruff check .
uv run mypy src
uv run pytest
```

**End-to-end, the real gate**

A `pytest` test that:

1. Serves `tests/fixtures/hcm/` over `http.server` on localhost, including a `robots.txt` that allows the crawl
2. Inserts a `crawl_sources` row pointing at it, `status = 'active'`
3. Runs `pipeline.run_source` against the live migrated database
4. Asserts `crawl_runs` finished `succeeded` with the expected metrics, and that `source_records` and `review_candidates` rows exist with correctly normalized VND prices, `+84` phones, and a resolved HCMC ward
5. Shells out to `go run ./cmd/reviewctl approve <id>`
6. Asserts the canonical `merchants` / `catalog_items` / `catalog_offers` rows now exist with `last_verified_at` set
7. Re-runs the pipeline against unchanged fixtures and asserts zero new candidates and a bumped `last_seen_at`, proving change detection works

Step 7 is what proves the freshness policy rather than merely claiming it.

**Manual smoke against a real audited source**

After the audit produces at least one permitted HCMC domain, run one live crawl with a low page cap and inspect the resulting candidates by hand before approving anything.

---

## Out of scope

- Mobile recommendation feed (`GET /v1/recommendations`) and the Expo screens that would consume it
- Ha Noi and Da Nang geography seeds and sources
- Google Places rating and review enrichment
- Image hosting or re-hosting of merchant photography
