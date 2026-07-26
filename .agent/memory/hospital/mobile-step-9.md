---
type: reference
tags: [mobile, expo, patient-app, step-9]
created: 2026-07-16
agent: main
---

Step 9 patient mobile application is implemented in `apps/mobile` as an Expo Router application. It includes Vietnamese-first and English UI, simulated/API OTP authentication, AsyncStorage persistence, five bottom tabs, family profiles, appointment discovery and booking, 10-minute MoMo/local payment states, local QR ticket and check-in, authenticated `/ws?token=...` updates with REST polling fallback, HomeCare booking and nurse tracking, laboratory and vaccination records, notifications, and hotline support.

The app uses Tam Anh tokens from `@hospital/design-tokens`, Barlow with Vietnamese glyph support, large accessible controls, seeded fallback data, and the generated OpenAPI types from `@hospital/api-client`. Added mobile dependencies are installed in `pnpm-lock.yaml` using the user-approved `npx --yes pnpm@11.13.0 install` because `pnpm` is not directly available in the shell.

Verification completed on 2026-07-16: mobile ESLint and TypeScript typecheck both pass. Commands emit a non-failing engine warning because the agent environment runs Node 26 while the repository requests Node 22.
