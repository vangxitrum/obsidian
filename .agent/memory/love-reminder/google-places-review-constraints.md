---
type: reference
tags: [google-maps, places-api, reviews, merchant-data]
created: 2026-07-26
agent: main
---

Google Places API (New) can enrich arbitrary merchant branches with `rating`, `userRatingCount`, `googleMapsUri`, and at most five relevance-sorted reviews. Rating fields use the Enterprise SKU; review text uses Enterprise + Atmosphere. Google Maps scraping is not an acceptable ingestion path.

Store the Google Place ID indefinitely, but treat other Places content as live or temporary API content subject to current caching restrictions. Display Google Maps attribution, distinguish Google content from first-party content, and provide required author attribution plus direct Google Maps links for review excerpts. Google Business Profile API can list all reviews only for locations authorized through the managing merchant account, so it cannot supply all reviews for arbitrary merchants.

**How to apply:** Map Place IDs to `merchant_locations`, fetch rating/count on demand with a minimal field mask, and initially link users to Google Maps rather than persisting review text. Keep this distinct from internal crawler review, which approves proposed merchant data before publication.
