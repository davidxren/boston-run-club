# Boston run club finder

Web app that helps someone in Boston find a run club and keep going to it. Not built yet. The product name is undecided and lives only in `NEXT_PUBLIC_APP_NAME`; never hardcode a name.

## Read first

`docs/build-session-v2.md` is the session contract: mission, hard rules, stack rulings, data model, deliverables D1 to D9 with acceptance conditions, gates, and the required report format. Read it fully before any work. Deliverables are done in order, one commit each, message `D<n>: <what>`.

`docs/handoff-2026-09-03.md` is project context and product thesis. `docs/verification-2026-09-03.md` is web research on the seeded clubs; it informs a later human pass and never changes seed data.

## Rules that never relax

- Nothing fabricated. No invented schedules, addresses, counts, screenshots or test output. Seed facts load as-is, `verified=false`.
- No 1:1 matching, messaging, friend requests, follows, swiping, recommendations of people, star ratings, streaks, badges, points, leaderboards. Not now, not proposed.
- The test harness is sacred. Never edit a test to make it pass, never skip, never mark flaky, never lower a threshold.
- Other users are visible only per Hard rule 7 in the contract. Default `showMeToOthers=false`.
- Missing data renders as missing with the exact string defined per field. Never inferred, never defaulted to a plausible value.
- Geolocation only on tap of "Near me". No third-party scripts except PostHog when its key is set.
- Every page works at 375px with no horizontal scroll.

## Stack, settled

Next.js 15 App Router, TypeScript strict, Tailwind, pnpm. Postgres via Prisma, `docker compose up db` locally. Auth.js v5 magic link with the Prisma adapter, console transport in dev. Leaflet + react-leaflet with CARTO tiles, no map keys. Haversine in app code. Vitest and Playwright (iPhone 13 emulation). Time zones via `Intl`, `date-fns` and `date-fns-tz` allowed. Any other runtime dependency: ask first.

## Commands

`pnpm install`, `pnpm db:migrate`, `pnpm db:seed`, `pnpm dev`, `pnpm test`, `pnpm test:e2e`, `pnpm lint`, `pnpm typecheck`. Env vars are listed in `.env.example`, one line each; keep it current when adding one.

## Craft

Domain names for modules and functions (`computeOccurrences`, `facesForUser`), no `utils` or `helpers`. No `any`, no bare catch, no dead or commented-out code. Comments say why. Structured JSON log events. Module ceiling 300 lines; argue exceptions in the report. Server components by default; client components only for the map, filters, and buttons with state.

## Copy and UI

Mobile first, black on white, one accent `#FF3B1F` for the primary button and the going count only. Sentence case. Verbs on buttons, nouns on labels. No emoji, no exclamation marks, no jokes in empty states, no em dashes anywhere in UI copy, docs or reports. Distances "0.8 mi", times "6:30 pm", dates "Wed Sep 9".

## Ask vs decide

Decide and list in the report: copy strings not fixed in the contract, filter defaults, component boundaries, Tailwind config, index choices, error page layout. Ask before: any schema field beyond the contract, any runtime dependency beyond the stack, any external network call other than CARTO tiles and Resend, any change to visibility rules, any edit to seeded club facts, any second accent color, any animation.

## Reports

End every session with the report format in the contract, in that order, nothing before item 1. Exact commands to reproduce every claim. "Works" and "clean" are not evidence.
