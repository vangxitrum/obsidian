---
type: decision
tags: [catalogue, doctors, homecare, mobile, seed]
created: 2026-07-26
agent: main
---

The synchronized API seed and mobile fallback catalogue contain 9 specialties, 20 doctors, and 12 HomeCare services. Added specialties are Nhi khoa, Nội tổng quát, Thần kinh, and Tim mạch. Doctor image URLs are intentionally empty and the appointment booking UI does not render doctor images.

Priced laboratory services supplied by the user are represented as selectable HomeCare services. Unpriced chronic-disease, infectious-disease, cancer-marker, and pregnancy test groups are described under the 150,000 VND home blood-collection service rather than assigning invented prices. The existing flu vaccination service remains because seeded vaccination records depend on it.

**Why:** Keep online and offline demo catalogues equivalent, preserve factual pricing, and avoid breaking the existing vaccination journey.

Regression coverage for catalogue size, representative names/prices, and empty doctor images is in `apps/mobile/__tests__/api.test.ts`.
