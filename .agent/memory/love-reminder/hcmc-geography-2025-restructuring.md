---
type: reference
tags: [geography, vietnam, hcmc, addresses, data]
created: 2026-08-06
agent: main
---

Vietnam's local government restructuring took effect 2025-07-01. Ho Chi Minh City absorbed Bình Dương and Bà Rịa - Vũng Tàu, the district tier was abolished, and the city now has 168 commune-level units: 113 phường, 54 xã, and the Côn Đảo đặc khu. Migration `000009_hcm_geography_seed` seeds all of them plus 1376 legacy aliases.

**No accessible source publishes the official GSO codes for the new units.** `danhmuchanhchinh.gso.gov.vn` was unreachable, and `dvhcvn/data` is still pre-2025 (63 provinces). The seed was generated from `vietmap-company/vietnam_administrative_address` (`admin_new` plus its old-to-new mapping workbook) and cross-checked against `phucanhle/vn-xaphuong-2025` and the list at xaydungchinhsach.chinhphu.vn; all three agree on all 168 names. So `administrative_areas.code` holds the reference dataset's identifier and `gso_code` is a separate nullable column, still NULL. Nothing depends on `gso_code`.

**Legacy district names must keep resolving.** Crawled merchant addresses will print "Quận 3" or "Bình Thạnh" for years. Every pre-2025 district and ward name that fed a new unit is in `administrative_area_aliases` with `source = 'legacy'`. One legacy district maps to many new wards on purpose, and the resolver treats a multi-hit alias as district-level context rather than a precise area match. Quận 1 resolves to Bến Thành, Cầu Ông Lãnh, Sài Gòn, and Tân Định; Quận 3 to Bàn Cờ, Xuân Hòa, and Nhiêu Lộc.

**Diacritic-stripped keys are genuinely ambiguous in HCMC.** "Xã Thạnh An" (Cần Giờ) and "Xã Thanh An" (Dầu Tiếng) both normalize and slugify to `thanh-an`. That is why areas are keyed by dataset code rather than slug, and why the resolver retries with the accented form before giving up.

**How to apply:** Regenerate the seed with `services/backend/scripts/hcm_seed_generator.py`, which asserts the 168-unit count and the cross-source name agreement before emitting SQL. Keep `crawler.normalize.text.normalize_area` byte-identical in behaviour to that script's normalizer, since seeded `normalized_alias` values are looked up with its output. Related: [[merchant-crawler-implementation]].
