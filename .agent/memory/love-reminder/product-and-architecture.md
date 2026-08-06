---
type: decision
tags: [product, architecture, mobile, notifications, crawler]
created: 2026-07-23
agent: main
---

Love Reminder is a Vietnamese-first thoughtful assistant for partners, family, and friends. It prioritizes meaningful actions over commerce and will later monetize clearly disclosed affiliate links from approved Vietnamese merchants.

The mobile app uses Expo SDK 54, React Native, Expo Router, and TypeScript under `apps/mobile`. The warm editorial design uses Lora headings, Be Vietnam Pro UI text, cream surfaces, plum primary actions, and terracotta/sage/gold accents. The mobile identity flow now uses Clerk passwordless email OTP with SecureStore-backed sessions; see [[clerk-authentication]]. The API client sends Clerk bearer tokens, but the Go backend still validates Supabase JWTs and therefore requires an identity migration before authenticated Clerk users can load production data. Expo notification registration and the 60-day local schedule reconciliation path remain in place. An explicit persisted `Xem bản mẫu` mode keeps local demo data available without silently masking production errors.

The Phase 1 backend is implemented in Go 1.26 under `services/backend`. Supabase provides email OTP authentication and PostgreSQL; Go owns API and business logic. River provides transactional, durable PostgreSQL-backed notification jobs. Expo push delivery persists tickets, checks receipts, disables invalid tokens, repairs interrupted occurrences, and expires stale work. The mobile app acknowledges exact locally scheduled occurrence IDs, and the worker suppresses push only for those device-occurrence pairs. Embedded application and River migrations run through `/migrate`; `/api` and `/worker` are separate binaries in one distroless image. Crawling remains a later phase restricted to approved Vietnamese merchants, using feeds/APIs first, Goquery for static pages, and Chromedp for approved JavaScript pages.

DigitalOcean App Platform in Singapore is the preferred initial deployment for the Go API, notification worker, and crawler jobs, targeting up to 10,000 users.

**Why:** This keeps the initial mobile experience coherent while separating reliable background work from the user API and retaining simple managed identity and data services.

**How to apply:** Preserve `apps/mobile` for Expo and `services/backend` for the API, worker, and migrations. Run River through a direct PostgreSQL connection or Supavisor session mode, not transaction pooling. This development host has no IPv6 route, so use the project's `ap-southeast-2` Supavisor session pooler rather than its IPv6-only direct database hostname. Production auth and push still require Supabase and EAS credentials; demo mode must remain explicit and must never mask production errors.
