---
type: fact
tags: [overview, nextjs, architecture]
created: 2026-07-09
agent: main
---

`skin-prod` is a simple Next.js app: a multi-step quiz (skin type, concerns, age range, sensitivity) that recommends skincare products. Built from an empty repo in one session (2026-07-09).

Stack: Next.js 16 App Router + TypeScript + Tailwind v4 + shadcn/ui ([[base-ui-shadcn-variant]] — it's Base UI under the hood, not Radix) + Prisma 7 + SQLite ([[prisma-7-driver-adapters]]).

Architecture decisions:
- Recommendation engine is DB-backed but does matching/scoring in application TypeScript, not SQL — `Product.skinTypes`/`Product.concerns` are comma-separated strings validated against a fixed vocabulary in `src/lib/constants.ts`, because SQLite has no native array/enum type in Prisma and a join table added ceremony without benefit at ~17 catalog rows. Matching logic lives entirely in `src/lib/recommend.ts`.
- Quiz state is intentionally stateless: the wizard (`src/components/quiz/QuizWizard.tsx`) holds answers in React state, then encodes them into `/results` URL query params on submit (no login, no localStorage, no server session). `src/lib/quiz-params.ts` parses/validates them back out.
- No auth, no payments, no admin UI — deliberately scoped down per user's "simple app for now" request.

Verified end-to-end with a throwaway Playwright driver script (chromium-cli wasn't available in this environment) — two full quiz runs with different answer combos produced visibly different, correctly-scored recommendation sets, zero console errors after fixing the Base UI warnings above.

Not yet committed to git as of end of session — repo has all files staged/untracked, user has not asked for a commit yet.

**2026-07-10 additions:**
- Product catalog replaced with 30 real products (name/brand/price) scraped from hasaki.vn category listing pages via headless Chromium (the site is a JS-rendered SPA, static fetch returns only header/nav — see how `prisma/seed.ts` sources data). skinTypes/concerns tags and all description text are original, not copied from the retailer. Prices are real VND, `Product.price` is `Int`, formatted with `Intl.NumberFormat('vi-VN', {style:'currency',currency:'VND'})` via `formatVnd()` in `src/lib/utils.ts`.
- Quiz gained two more steps: a "personal info" step (name + optional email) now FIRST in the wizard (`src/components/quiz/steps/PersonalInfoStep.tsx`), and a "budget" step (under-200k/200k-500k/over-500k/any) last, scored (not hard-filtered) in `recommend.ts`. Results page greets by name if provided.
- Added `motion` (the renamed/current Framer Motion package, NOT `framer-motion` — that's the old name) for step-transition and results-grid stagger animations. React 19 compatible.
- Added a post-quiz "AnalyzingScreen" (`src/components/quiz/AnalyzingScreen.tsx`) that shows ~2.2s of cycling status text ("Analyzing your skin profile...", etc.) before navigating to `/results`. **This is purely cosmetic UX theater the user explicitly asked for ("show that we may integrate with AI") — there is no real AI call.** If a future task is "wire up the real AI," this loading screen already exists and just needs the actual backend call slotted in before `onDone()` fires.
- Verified mobile responsiveness via Playwright's `devices['iPhone 12']` viewport — confirmed zero horizontal overflow once entrance animations settle (a raw screenshot taken immediately after `waitForSelector` can catch mid-animation frames that look like overflow but aren't; wait ~500-600ms after the element appears before trusting a mobile screenshot).
- User asked to switch the catalog source to guardian.com.vn — **guardian.com.vn actively blocks automated browsers with a Cloudflare "Just a moment..." challenge (HTTP 403), even after waiting it out.** Did not attempt to bypass it (no stealth/proxy/CAPTCHA workarounds). Stayed on hasaki.vn as the data source per user's choice.
- Added real product photos as `Product.imageUrl`: hotlinked directly to hasaki.vn's own CDN (`media.hcdn.vn/...`), NOT downloaded/rehosted — copying and storing their product photography would be reproducing copyrighted images, whereas linking to their own hosted URL isn't. hasaki.vn lazy-loads images (native `loading="lazy"`, placeholder `data:image/gif` src swapped in by the browser on scroll-into-view, no `data-src` attribute to read directly) — extracting real `src` values required scrolling the category page through in ~700px increments before reading `img.currentSrc`, not just reading attributes off a static DOM snapshot. `ProductCard.tsx` renders the hotlinked `<img>` with an `onError` fallback back to the category-label placeholder block if a URL ever breaks.
