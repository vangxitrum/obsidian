---
title: Merchant Catalog Data Architecture
date: 2026-07-26
tags:
  - project/love-reminder
  - architecture
  - data
  - crawler
status: planned
---

# Merchant Catalog Data Architecture

V1 covers gifts and experiences in HN, HCM, and DN. Sources must be permission-compatible. Purchases and bookings redirect to merchants. Crawler records require review before publication.

## Data Model

![[diagrams/merchant-catalog-erd.drawio.png]]

## Crawl and Publication Flow

![[diagrams/crawl-publication-flow.drawio.png]]

## Source

[[diagrams/merchant-catalog-data-model.drawio|Open the editable two-page Draw.io diagram]]

## Key Decisions

- Keep current city boundaries while modeling merchant service areas explicitly.
- Support merchant-wide, location-specific, item-specific, and mixed promotions.
- Separate canonical typed tables from source records.
- Preserve raw addresses and source provenance.
