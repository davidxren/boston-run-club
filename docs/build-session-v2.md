# Boston run club finder. Build session v2

Changes from v1, Sept 3, 2026. Strike any of these before starting and the rest still hold.

1. Bug: `/boston/run-clubs/[neighborhood]` and `/boston/run-clubs/[weekday]` were two dynamic segments at one path level, which does not build. Now one `[segment]` route that resolves weekday first, then neighborhood. D8.
2. Bug: the occurrence window was the next 14 days, which excludes the 36 hour check-in window and yesterday's runs. Now `now - 36h` to `now + 14d`. Data model rules.
3. Bug: `X-Test-Now` was gated on `NODE_ENV=test`, which Next.js overrides. Now gated on `TEST_HOOKS=1`. Stack, D5.
4. Bug: Playwright read the magic link from dev server stdout. Now from `.tmp/last-signin-link.txt`. Stack, D4.
5. Mission said 7 days, everything else said 14. Now 14 everywhere.
6. OG images were "at build time", which goes stale as soon as the next occurrence changes. Now a per-club `opengraph-image.tsx` with `revalidate = 3600`. D8.
7. In-memory rate limit is per instance and useless on serverless. Kept for this session, documented as a known gap. D7.
8. DST acceptance test on `computeOccurrences`. D2.
9. Verified clubs whose `lastCheckedAt` is older than 60 days render "Verified <date>, may be out of date". D3.
10. Ruling on "arrive together": club opt-in via the claimer. One optional string, `Session.firstTimerMeet`, rendered under the occurrence. No greeter role, nothing 1:1. Schema, D3, D7.
11. PWA manifest so the site can be added to a home screen. No service worker. D8.
12. Seed data moved out of this document into `seed/boston-run-clubs.json` and `seed/boston-neighborhoods.json`. The clubs file has 43 clubs, not 44 as the handoff says. D1, Seed data.
13. Time zone math: platform `Intl` APIs. `date-fns` and `date-fns-tz` allowed if needed, nothing else. Stack.

Product name is NOT DECIDED. It is config only: `NEXT_PUBLIC_APP_NAME`, default `PLACEHOLDER`. Every user-facing surface reads from that variable. Renaming later is a one-line change. Do not hardcode a name anywhere.

## Mission

A person in Boston opens the site on their phone, taps "Near me", sees run clubs meeting in the next 14 days sorted by distance, opens one, reads exactly where to stand and what will happen on a first visit, taps "I'm going" with a first-timer flag, and after the run taps "I went" and sees who they have now run with more than once.

## Context (read once, do not restate in the report)

- Inspiration: Cluby, an NYC private beta (Sept 2026) for discovering run clubs, book clubs, supper clubs and niche communities. This is the Boston version, running only for now, built to be better on the one thing that matters: what it is actually like to show up.
- Thesis: friendships form by showing up to the same thing repeatedly with the same people, not by matching. Friend-matching apps feel like blind dates. So: no swiping, no 1:1 matching, no DMs. The product does two things. It removes first-visit uncertainty (where to stand, what happens, how fast, who shows up, what after, what it costs). It makes repeat attendance visible (you have been 3 times; you have run with these 4 people twice or more).
- Existing products, for positioning only. Do not copy their UI. Fit Scene Boston (editorial guide, 70+ clubs, no per-session data), SweatPals Boston (event feed), Strava clubs, Meetup, Heylo (club management tool many clubs use), RunClub app, Run Clubs App, Paka Run Club Finder, Wylder, Stride Club, Train Pack. None of them answers "what is the first time like" per club, none shows first-timer counts per session, none shows who you have repeatedly run with.
- The current best sources for Boston club facts are Instagram accounts and the Fit Scene guide. Seed data in `seed/` was compiled from the Fit Scene guide dated 2026-06-02 plus scattered listings. `docs/verification-2026-09-03.md` records what the open web says about each club as of Sept 3, 2026; it is input for the human verification pass, not for the seed. Treat every seeded fact as unverified.

## Hard rules

1. Nothing fabricated. No invented schedules, addresses, meet spots, counts, screenshots or test output. Seed data ships with `verified=false` and a `sourceUrl`. The UI shows "Unverified. Last checked Sep 2, 2026" on every club until `verified=true`.
2. No 1:1 matching, no messaging, no friend requests, no follows, no swiping, no "people you may like". Not in this session and not proposed for later.
3. Every page renders at 375px width with no horizontal scroll. Lighthouse mobile performance >= 90 on `/boston/run-clubs` and on three club pages.
4. Meet spots render as circles sized by `precision`; a pin only when `precision=exact`.
5. The test harness is sacred. Never edit a test to make it pass, never skip or mark flaky, never lower a threshold.
6. Geolocation is requested only on tap of "Near me". Never on load.
7. Other attendees' names are visible only to a user who has a check-in at the same club, only for people who share 2+ checked-in occurrences at that club with them, and only when both users have `showMeToOthers=true`. Default is false. Nothing else about a user is ever visible to another user.
8. No third-party scripts except PostHog, and PostHog only when `NEXT_PUBLIC_POSTHOG_KEY` is set.
9. Missing data is shown as missing with the exact string defined per field below. Never inferred, never defaulted to a plausible value.
10. Commit per deliverable, message `D<n>: <what>`. No diff outside the current deliverable.

## Non-goals (this session)

- Native iOS or Android, push notifications
- Club types other than running. `clubType` exists in the schema; validation accepts only `running`
- GPS run tracking, Strava activity sync, routes, pace calculators
- Payments, memberships, waivers, ticketing
- Chat, DMs, comments, user photo uploads. Club logos by URL only
- Star ratings or numeric scores of any kind. Vibe tags only
- Any city other than Boston. `citySlug` exists; only `boston` is seeded
- Gamification: streaks, badges, points, leaderboards
- Any redesign. UI direction below is fixed for this session

## Stack (rulings, do not relitigate)

- Next.js 15 App Router, TypeScript `strict`, Tailwind, pnpm
- Postgres via Prisma. Local: `docker compose up db` with `postgres:16`. Prod: any `DATABASE_URL`
- Auth: email magic link via Auth.js v5 with the Prisma adapter. Email transport: Resend when `RESEND_API_KEY` is set. When it is not set and `NODE_ENV!=production`, `EMAIL_TRANSPORT=console` prints the sign-in link to stdout. A production build claim without `RESEND_API_KEY` is a failed gate, not a fallback
- Test hooks: `TEST_HOOKS=1` enables exactly two things. The `X-Test-Now` request header overrides the app clock, and the console email transport also overwrites `.tmp/last-signin-link.txt` with the most recent sign-in link. Both are ignored when `NODE_ENV=production`. `pnpm test:e2e` sets `TEST_HOOKS=1` and starts the server itself. `.tmp/` is gitignored
- Time zones: platform `Intl` APIs. `date-fns` and `date-fns-tz` are allowed if needed. No other date library
- Map: Leaflet + react-leaflet, CARTO light basemap tiles. No Mapbox, no Google Maps, no API keys
- Distance: haversine in app code. No PostGIS this session
- Tests: Vitest for units and route handlers, Playwright (iPhone 13 emulation) for the five e2e flows named in the deliverables
- `pnpm lint` (eslint + prettier) and `pnpm typecheck` with zero warnings

## Data model (Prisma, exact)

```
City         id, slug @unique ("boston"), name, centerLat, centerLng, timezone ("America/New_York")
Neighborhood id, citySlug, slug, name, lat, lng            @@unique([citySlug, slug])
Club         id, slug @unique, name, clubType Enum(running), citySlug, neighborhoodSlug?,
             category Enum(national, brand_gym, identity, social, classic, unknown),
             summary String? (<=160 chars, plain text),
             instagram?, website?, stravaClubId?, heyloUrl?,
             cost Enum(free, dues, dropin, unknown), costNote?,
             pacePolicy Enum(no_drop, pace_groups, single_pace, unknown), paceNote?,
             distanceNote?, sizeTypical Enum(under15, from15to50, over50, unknown),
             identityNote?, socialAfter?, bagDrop Enum(yes, no, unknown), waiver Enum(yes, no, unknown),
             transit?, scheduleStatus Enum(seeded, check_instagram, unknown),
             verified Boolean @default(false), sourceUrl, lastCheckedAt DateTime,
             claimedByUserId?, status Enum(active, paused, unknown, pending_review) @default(unknown)
Session      id, clubId, weekday Int (0=Sun..6=Sat), startLocal String ("HH:MM" 24h), durationMin?,
             cadence Enum(weekly, biweekly, monthly, irregular), seasonNote?,
             meetSpotName, meetSpotDetail? (where exactly to stand), lat, lng,
             precision Enum(exact, block, neighborhood, city), active Boolean @default(true),
             firstTimerMeet String? (<=120 chars, set only by a claimer or admin)
FirstTimeGuide id, clubId @unique, arriveByMin Int?, lookFor?, whatHappens Json? (string[]),
             whoShowsUp?, howFast?, whatToBring?, afterwards?, updatedByUserId?, updatedAt
User         id, email @unique, displayName?, homeNeighborhoodSlug?, showMeToOthers Boolean @default(false), createdAt
Going        id, userId, sessionId, occursOn Date, firstTime Boolean, createdAt   @@unique([userId, sessionId, occursOn])
Checkin      id, userId, sessionId, occursOn Date, firstTime Boolean, createdAt   @@unique([userId, sessionId, occursOn])
VibeTag      id, slug @unique, label
VibeVote     id, userId, clubId, vibeTagId, createdAt                             @@unique([userId, clubId, vibeTagId])
ClubSubmission id, clubId?, payload Json, submitterEmail, status Enum(pending, approved, rejected), createdAt, reviewedAt?
```

Rules on top of the schema:
- Occurrences (a session on a concrete date) are computed from `Session` in `America/New_York` for the window `now - 36h` to `now + 14d`. Never stored. `computeOccurrences(session, window, now)` is the single implementation; nothing else derives dates from `weekday` and `startLocal`. Directory cards and the club page occurrence list show only occurrences with start >= now. D5 uses the past part of the window.
- `precision` radius on the map: exact = pin, block = 150 m circle, neighborhood = 600 m circle, city = 2500 m circle with the label "Location varies, check Instagram".
- `VibeVote` requires >= 1 `Checkin` by that user at that club. Max 5 tags per user per club.
- Admins are the emails in `ADMIN_EMAILS` (comma separated). No admin flag in the DB.
- Vibe tags seeded, in this order: solo_friendly "Fine to come alone", no_drop "Nobody gets dropped", beginner_ok "Beginners welcome", easy_pace "Easy pace", fast "Fast", chatty "Chatty", quiet "Quiet", small_group "Small group", big_group "Big group", social_after "Hangs out after", early_morning "Early morning", evening "Evening", women_led "Women led", lgbtq_inclusive "LGBTQ+ inclusive", bipoc_led "BIPOC led", dogs_ok "Dogs ok", strollers_ok "Strollers ok".

## Product principles (apply to every micro-decision)

1. The unit is the session, not the person.
2. Social proof over matching. Show counts ("6 going, 2 first-timers"). Never suggest a person.
3. Repetition over introduction. "You have run with Maya 3 times" appears only after the fact, never before.
4. Zero first-visit uncertainty. On a 375px screen, before scrolling, a club page answers: when next, where to stand, how fast, what it costs.
5. Missing is shown as missing. Exact empty strings per field: pacePolicy unknown = "Pace policy: unknown. Been? Tell us." cost unknown = "Cost: unknown." sessions empty = "Schedule not confirmed. Check their Instagram." FirstTimeGuide field empty = "Not filled in yet. Been? Tell us." Each of these links to the suggest-an-edit form for that club.

## UI direction (fixed)

Mobile first. Black text on white. One accent, `#FF3B1F`, used only for the primary button and the going count. System font stack, 17px base, 1.4 line height, 8px spacing grid. Cards separated by 1px rules. No shadows, no border radius above 6px, no gradients, no illustrations, no icons except a 16px map pin and a 16px chevron. Sentence case everywhere. Verbs on buttons: "I'm going", "I went", "Suggest an edit", "Claim this club", "Near me". Nouns on labels. No emoji, no exclamation marks, no "Oops", no jokes in empty states. Loading: skeleton rows matching final layout. Errors: one sentence, what happened, what to do. Distances in miles to one decimal, times as "6:30 pm", dates as "Wed Sep 9". Dark mode: not this session.

## Deliverables

### D1. Scaffold, schema, seed
- `pnpm install && pnpm db:migrate && pnpm db:seed && pnpm dev` works from a clean clone with only `DATABASE_URL` set.
- `pnpm db:seed` is idempotent (upsert on slug) and prints exactly `seeded: clubs=<n> sessions=<m> neighborhoods=<k> tags=17`.
- Seed reads `seed/boston-run-clubs.json` and `seed/boston-neighborhoods.json`. Both files are already in the repo. Do not retype, reorder, reformat or edit their contents. File shapes are in the Seed data section.
- Acceptance: Vitest test `seed.integrity` asserts every club has slug, name, sourceUrl, lastCheckedAt, category; every session has lat, lng, precision, weekday in 0..6, startLocal matching `^\d{2}:\d{2}$`; every club with `scheduleStatus=seeded` has >= 1 session; every club with `scheduleStatus!=seeded` has 0 sessions.

### D2. Directory `/boston/run-clubs`
- Default list view, toggle to map view. Card: name, neighborhood, next occurrence ("Wed 6:30 pm, Dorchester" or the exact empty string), pace policy, cost, "Unverified" tag when `verified=false`.
- Filters live in URL query params and survive reload: `day` (multi, 0..6), `time` (morning = start < 10:00, midday = 10:00 to 16:59, evening = start >= 17:00), `hood` (multi slug), `pace`, `cost`, `tag` (vibe slug), `first` (=1 means club has a FirstTimeGuide with `whatHappens` filled and pacePolicy in {no_drop, pace_groups}).
- "Near me": requests geolocation on tap. Sorts by haversine distance from the user to the club's next occurrence meet spot. Shows "0.8 mi" on each card. Permission denied or unavailable: sort stays default, a neighborhood picker opens, no error modal.
- Default sort without location: next occurrence soonest first; clubs with no sessions last, alphabetical.
- Acceptance: Playwright `directory.spec`: load, set day=3 and time=evening via UI, card count equals `GET /api/clubs?day=3&time=evening` count, URL contains both params, reload preserves both and the count.
- Acceptance: Vitest `occurrences`: (a) a weekly session with `weekday=3`, `startLocal="18:30"`, `now=2026-09-03T12:00:00-04:00` yields exactly two future occurrences, Sep 9 and Sep 16, in that order. (b) The instant for local `06:30` is `2026-10-31T10:30:00Z` on Sat Oct 31, `2026-11-01T11:30:00Z` on Sun Nov 1, `2027-03-13T11:30:00Z` on Sat Mar 13, `2027-03-14T10:30:00Z` on Sun Mar 14, and all four render as "6:30 am". (c) With `now=2026-09-03T12:00:00-04:00`, a `weekday=3` `startLocal="06:30"` session includes Sep 2 06:30 as a past occurrence inside the 36 hour window, and a `weekday=2` `startLocal="06:30"` session does not include Sep 1. (d) Sessions with cadence `biweekly`, `monthly` or `irregular` produce no occurrences this session (the schema has no anchor date); their cards show `seasonNote` text in place of the time, same as an inactive session.

### D3. Club page `/clubs/[slug]`
- Above the fold at 375px: name, next 3 occurrences each with meet spot name and an "I'm going" button, pace policy line, cost line, a "First time here?" jump link.
- An occurrence row shows "First-timers: <firstTimerMeet>" under the meet spot name when the session has `firstTimerMeet`, nothing when it is null.
- First time section, in this order: Arrive by, Look for, What happens (numbered), Who shows up, How fast, What to bring, Afterwards. Empty fields render the exact empty string and link to `/clubs/[slug]/suggest`.
- Map with circle or pin per `precision`.
- Vibe tags with vote counts, descending, max 8 shown, "Show all" reveals the rest.
- Facts block: transit, bag drop, waiver, size, cost note, pace note, distance note, identity note, social after. Unknown values render "unknown", never hidden.
- Footer: `verified=false` renders "Unverified. Last checked <date>". `verified=true` with `lastCheckedAt` 60 days old or less renders "Verified <date>". `verified=true` older than 60 days renders "Verified <date>, may be out of date". Then source link, "Suggest an edit", "Claim this club". Vitest `verification-label`: the three strings, with the boundary at exactly 60 and 61 days.
- Acceptance: Vitest `club-pages` requests `/clubs/<slug>` for every seeded slug through the route handler and asserts 200 and the club name in the HTML. Lighthouse mobile performance >= 90 on three club pages, chosen as: the club with the most sessions, one with `scheduleStatus=check_instagram`, one at random. Paste the three scores in the report.

### D4. Auth and Going
- Magic link sign in at `/signin`. First sign in requires `displayName` (2..40 chars). `homeNeighborhoodSlug` optional. `showMeToOthers` toggle, default off, with this exact helper text: "When on, people who have checked in to the same club twice or more with you can see your display name and how many times you have run together. Nothing else. When off, nobody sees you."
- "I'm going" on an occurrence creates `Going(occursOn)`. Tapping again deletes it. A "First time at this club" checkbox is shown only when the user has zero `Checkin` at that club.
- Occurrence shows "<n> going" and "<f> first-timers" (hidden when 0). Names are never shown on occurrences.
- Acceptance: Playwright `going.spec` with `EMAIL_TRANSPORT=console` and `TEST_HOOKS=1`: sign in user A (read link from `.tmp/last-signin-link.txt`), mark going with first-timer on the first occurrence of the seeded club with the most sessions, expect "1 going" and "1 first-timer"; sign in user B in a second context, mark going, expect "2 going"; sign out both, counts persist on reload.

### D5. Check-in, history, faces
- From an occurrence's start time until 36 hours after, "I went" appears on the club page and on `/me` for occurrences the user marked going, plus a "Went somewhere else?" picker on `/me` listing today's and yesterday's occurrences citywide. Creates `Checkin`. Removes the matching `Going` if present.
- `/me`: clubs attended with counts, exact format "Pioneers Run Crew: 3 times". Sorted by count desc. No streaks, no badges.
- Faces: on `/me` per club and on the club page (only when the user has >= 1 `Checkin` there), list users sharing >= 2 checked-in occurrences at that club with the current user, where both have `showMeToOthers=true`. Section heading is the exact string "Ran with". Each row: `displayName` and "ran together 3 times". Nothing tappable.
- Acceptance: Vitest `faces.query` with fixtures: A and B share 2 occurrences, both opted in, B shown to A and A to B; A and C share 1, C not shown; A and D share 3 but D opted out, D not shown and A not shown to D; A opted out, sees nobody. Playwright `checkin.spec`: user A marks going, test advances the app clock via the `X-Test-Now` header (accepted only when `TEST_HOOKS=1` and `NODE_ENV!=production`), "I went" appears, tap, `/me` shows "<club>: 1 time".

### D6. Vibe votes
- On the club page, after >= 1 `Checkin` there, tags become tappable toggles. Max 5 per user per club. Counts update in place.
- Acceptance: Vitest on `POST /api/clubs/[slug]/vibes`: without checkin returns 403 `{ "error": "checkin_required" }`; the 6th distinct tag returns 400 `{ "error": "max_5" }`; toggling off deletes the row.

### D7. Submit, suggest an edit, claim, admin
- `/submit`: public form creating `ClubSubmission(pending)`. Fields mirror Club plus one Session plus FirstTimeGuide. Honeypot field. Rate limit 5 per hour per IP. In-memory this session; it is per instance and does nothing useful on serverless, and the README lists it as a known gap.
- `/clubs/[slug]/suggest`: same form prefilled, creates `ClubSubmission` with `clubId`.
- "Claim this club": signed-in user submits Instagram handle and a sentence. Admin approves, sets `claimedByUserId`. A claimer edits club, sessions (including `firstTimerMeet`) and guide directly at `/clubs/[slug]/edit`, and sets `verified=true` with `lastCheckedAt=now`.
- `/admin`: `ADMIN_EMAILS` only. Lists pending submissions, diff view against current club when `clubId` set, approve applies payload, reject stores `reviewedAt`.
- Acceptance: Playwright `submit.spec`: submit a new club with one session, sign in as admin, approve, `/clubs/<new-slug>` returns 200 with the name and the session in the next occurrences. Non-admin GET `/admin` returns 404.

### D8. SEO and share
- Routes: `/boston/run-clubs`, `/boston/run-clubs/[segment]`, `/clubs/[slug]`. `[segment]` is a single dynamic route: if the value is one of `sunday`..`saturday` it renders the weekday page, else if it matches a neighborhood slug it renders the neighborhood page, else 404. `seed.integrity` also asserts no neighborhood slug equals a weekday slug. Each page has title, meta description under 155 chars, canonical, and JSON-LD (`SportsOrganization` for a club, `Event` per occurrence with `startDate`, `location`, `isAccessibleForFree` when cost=free).
- OG image per club from `app/clubs/[slug]/opengraph-image.tsx`, generated on request with `revalidate = 3600`, never at build time: black on white, club name, next occurrence, neighborhood, app name. Text only.
- `manifest.webmanifest`: `name` and `short_name` from `NEXT_PUBLIC_APP_NAME`, `start_url` `/boston/run-clubs`, `display` `standalone`, `background_color` and `theme_color` `#FFFFFF`, one 512x512 PNG icon generated at build: the first letter of the app name, black on white, system font. No service worker. Acceptance: `curl -s localhost:3000/manifest.webmanifest | jq -r .name` prints the env value.
- `sitemap.xml` and `robots.txt`.
- Acceptance: `curl -s localhost:3000/sitemap.xml | grep -c "<loc>"` >= clubs + neighborhoods + 7 + 1. Paste the number.

### D9. Ops and docs
- `README.md`: setup, every env var with one line each, seed, deploy (Vercel + Neon), how a maintainer verifies a club (checklist: Instagram checked, meet spot confirmed, precision set, verified flag, lastCheckedAt).
- `pnpm test`, `pnpm test:e2e`, `pnpm lint`, `pnpm typecheck` all pass.

## Gates

- G0 before D1: node >= 20, pnpm, docker (or a reachable `DATABASE_URL`), `pnpm exec playwright install chromium` succeeds. Else STOP.
- G1 before D4: `AUTH_SECRET` set, `DATABASE_URL` reachable. `RESEND_API_KEY` optional in dev. Any statement of production readiness requires it.
- G2 before D8 OG images: `next/og` renders in this environment. Else render an SVG text card and say so in the report.
- On a failed gate print exactly `GATE <id> FAILED: missing <var or binary>`, write the report for what completed, stop. Do not stub past a gate.

## Ask vs decide

Decide and list in the report: copy strings not fixed above, filter defaults, component boundaries, Tailwind config, index choices, error page layout.

Ask before: any schema field beyond the model above, any runtime dependency beyond the stack list, any external network call other than CARTO tiles and Resend, any change to Hard rule 7 visibility, any edit to seeded club facts beyond loading them, any second accent color, any animation.

## Craft

Domain names (`computeOccurrences`, `facesForUser`, not `utils`, not `helpers`). No dead code, no commented-out code, no `any`, no bare catch, no `catch {}`. Comments say why. Logs are structured JSON events. Module ceiling 300 lines, argue exceptions in the report. Server components by default; client components only for the map, filters, buttons with state.

## Report format (in this order, nothing before item 1)

1. Test summary line: unit passed/failed, e2e passed/failed, lint warnings, typecheck errors.
2. Per deliverable: evidence and the exact commands to reproduce it.
3. Lighthouse scores for the four pages, with the command used.
4. Decisions taken without asking, each with the reason.
5. Not done, with the gate or blocker.
6. Open rulings needed, as questions.
7. Seed data status: count of clubs by `scheduleStatus`, and the list of clubs with `scheduleStatus!=seeded` by name.

## Seed data

Two files, already in the repo. Never retype them.

`seed/boston-run-clubs.json`: object with `schemaVersion`, `compiledOn`, `note`, `meetSpots` and `clubs`. `meetSpots` maps a key to `name`, `neighborhood` (slug or null), `lat`, `lng`, `precision`. All coordinates are approximate to the stated precision, none verified. `clubs` is an array of 43 clubs; each `sessions[].meetSpot` is a key into `meetSpots`, and the loader copies that spot's `name`, `lat`, `lng`, `precision` onto the Session, with `meetSpotDetail` from the session when present.

`seed/boston-neighborhoods.json`: object with `schemaVersion` and `neighborhoods`, an array of 12 with `citySlug`, `slug`, `name`, `lat`, `lng`.

Field defaults when omitted: `clubType=running`, `citySlug=boston`, `cost=unknown`, `pacePolicy=unknown`, `sizeTypical=unknown`, `bagDrop=unknown`, `waiver=unknown`, `verified=false`, `status=unknown`, `lastCheckedAt=2026-09-02`, `sourceUrl=https://fitscene.co/news/boston-run-clubs-2026`. `scheduleStatus=seeded` when `sessions` is non-empty, else the value given. Weekday: 0 Sun, 1 Mon, 2 Tue, 3 Wed, 4 Thu, 5 Fri, 6 Sat. `cadence=weekly` unless given. `instagram` is the handle without @.

Rules for loading: a session with `startLocal: null` is stored with `startLocal="00:00"` and `active=false`, and the directory shows the club's `seasonNote` text in place of the time. A club whose sessions are all inactive is treated as `scheduleStatus=check_instagram` for display. `sourceUrl="unverified"` is stored as-is and displayed as "No source yet".

## After this session (not now, for the report's open rulings only)

- Verification pass: one maintainer opens every Instagram account, fixes times and meet spots, sets `precision`, flips `verified`.
- First-time guides for the 10 clubs with the most going counts, written from actually attending.
- "Arrive together": ruled in v2. Club opt-in only, via a claimer setting `Session.firstTimerMeet`. No greeter role, no huddle feature beyond that string.
- Second club type. Schema is ready; validation is the only change.
