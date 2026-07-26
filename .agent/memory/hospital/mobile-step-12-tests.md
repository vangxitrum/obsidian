---
type: reference
tags: [mobile, testing, jest, maestro, step-12]
created: 2026-07-16
agent: main
---

Mobile Step 12 test/config work is complete under `apps/mobile` only. React Native Testing Library is pinned to `13.3.3` for compatibility with Jest 29/Expo 57, and the mobile TypeScript config includes Jest types. Four suites cover OTP success/failure, login language switching, payment countdown and failure status, accessible disabled primary controls, and AsyncStorage hydration/persistence. Two Maestro YAML flows cover the appointment/queue and HomeCare scripts using existing visible labels and accessibility metadata.

Verification on 2026-07-16: mobile lint passed, typecheck passed, and Jest passed 6 tests in 4 suites. Maestro was intentionally not run because the CLI/device are unavailable. Dependencies were installed with `npx --yes pnpm@11.13.0 --filter @hospital/mobile add -D @testing-library/react-native@13.3.3 --lockfile=false`, preserving the root lockfile. No production source was changed.

Step 13 corrections make the payment countdown test deterministic by advancing fake timers and processing asynchronous UI updates inside React `act`, without increasing timeouts. The mobile build now exports iOS to `apps/mobile/dist/ios` and Android to `apps/mobile/dist/android` instead of exporting web. Under transient Node `v22.23.1`, mobile lint, typecheck, all 6 Jest tests, and both native Expo exports pass.
