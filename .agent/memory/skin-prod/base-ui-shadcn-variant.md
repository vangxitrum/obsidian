---
type: fact
tags: [nextjs, shadcn, base-ui, react, components]
created: 2026-07-09
agent: main
---

This project's `npx shadcn@latest init` pulled a **Base UI (`@base-ui/react`)** variant, not the classic Radix-based shadcn/ui. Training data and most tutorials assume Radix — the API differs in ways that silently break or warn:

- No `asChild` prop. Use `render={<Link href="..." />}` on components like `Button` to compose with a custom element (Base UI's composition pattern, see `node_modules/@base-ui/react/docs/react/handbook/composition.md`).
- `Button` defaults to `nativeButton={true}` (expects the rendered element to be a real `<button>`). When composing with `render={<Link .../>}` (renders an `<a>`), also pass `nativeButton={false}` or Base UI logs a console warning on every render.
- `RadioGroup`/`Checkbox` use `value`/`onValueChange` and `checked`/`onCheckedChange` (same shape as Radix), but `RadioGroup`'s `value` must be a defined string from first render (e.g. `value ?? ""`, not `undefined`) — otherwise React logs an uncontrolled→controlled warning the moment the first option is picked.
- `Radio`/`Checkbox` render a visually-hidden native `<input>` (aria-hidden, clip-path hidden) alongside a visible `<span role="radio">`/`role="checkbox"`. The `id` prop passed to `RadioGroupItem` lands on the hidden input, not the visible span, so `getByLabel(...)` in Playwright/RTL tests resolves ambiguously (matches both elements). Prefer `getByRole('radio'|'checkbox', {name})` instead.

**Why:** hit all of the above building the quiz wizard (`src/components/quiz/*`) — cost a full debug/verify cycle to work out the warnings weren't in the original Radix-based shadcn docs.

**How to apply:** any future shadcn/ui component work in this repo (`skin-prod`) should assume Base UI semantics first, and check `node_modules/@base-ui/react/<component>/*.d.ts` before guessing at prop names from Radix-era muscle memory.
