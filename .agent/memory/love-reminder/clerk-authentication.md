---
type: decision
tags: [authentication, clerk, expo, mobile]
created: 2026-07-26
agent: main
---

The Expo mobile app is initialized against Clerk application `app_3H2vH0aI6VQnL1gwKeBNj0RRN6n` using `@clerk/expo` 4.1.0. `ClerkProvider` uses Clerk's SecureStore-backed `tokenCache`. The existing Vietnamese screens now provide separate passwordless email-code sign-in and sign-up flows, while explicit demo mode remains available. The settings screen recognizes the Clerk user and provides sign-out even when application data fails to load.

The mobile API client obtains bearer tokens from Clerk. The Go backend validates RS256 Clerk session tokens against `CLERK_ISSUER_URL`, optionally validates `azp` against `CLERK_AUTHORIZED_PARTIES`, and accepts only `user_*` subjects. Migration `000004_auth_identities` maps `(provider, subject)` to internal UUID profiles; first-use provisioning is transactionally serialized with a PostgreSQL advisory lock. Clerk token email is optional and enriches the profile when available.

The development API in tmux `love-reminder:api` runs on port `18080` with authentication enabled and issuer `https://climbing-satyr-78.clerk.accounts.dev`. Both local and tunneled health checks pass, and unauthenticated `/v1/me` requests return 401.

The ignored `apps/mobile/.env.local` contains only the API URL and `EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY`. A previously present `CLERK_SECRET_KEY` entry was removed without inspecting its value; server secrets must never be added to client code or committed.

**Why:** Clerk becomes the mobile identity provider without discarding the existing branded OTP and demo experiences.

**How to apply:** Preserve the `@clerk/expo` provider, token cache, and UUID identity mapping. Configure the backend issuer, run migrations before deployment, and complete a real sign-up plus session-restart test before production.
