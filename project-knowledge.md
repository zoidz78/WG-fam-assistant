# WG Family Assistant — Project Knowledge & Setup Guide

**Read this file first in any new chat on this project.** It's the single source
of truth for how this project works, what's built vs. designed-but-not-yet-wired,
and what to do next.

## What this project is

A household hub for the family + their live-in helper. Static site, hosted on
**GitHub Pages** (a separate GitHub account from `wg-groupbuy`, uploaded manually
by the user — this session has no push access to it). No build step, no backend
(yet) — plain HTML/CSS/JS files plus a couple of JSON data files.

**Pages so far:**
- `index.html` — Home Hub landing page. Reads `hub-cards.json` for what cards to
  show; only one exists today (Meals), but more are coming (chores, groceries,
  etc.) and should be added purely via that JSON file, never by hardcoding a new
  `<a class="hub-card">` block into this HTML.
- `meal-dashboard.html` — "What's Cooking": a weekly meal planner (breakfast/
  lunch/dinner, each split into an Adults card and a paired Kids card) plus a
  translated message thread per meal slot between the user and the helper.

**Sibling project (`wg-groupbuy`) established the patterns this one follows** —
generic HTML that never hardcodes a group/round's data, JSON files for static
reference data, Firestore for anything live/cross-device, a `firestore-rules.md`
alongside the code, and this same "read me first" doc format. When in doubt about
a design question not covered below, check how `wg-groupbuy-project-knowledge.md`
handled the analogous case before improvising.

## Files

| File | Purpose |
|---|---|
| `index.html` | Home Hub landing page. Fetches `hub-cards.json` and renders one real card per entry (plus a hardcoded dashed "ghost" placeholder card, always last). |
| `hub-cards.json` | List of Home Hub cards: `{id, href, icon, text: {en, zh, tl: {tab, title, desc}}}`. Add an entry to add a new section — no HTML/JS change needed. |
| `meal-dashboard.html` | The meal planner + message thread. Fetches `recipes.json` at boot. Computes "this week" (Monday–Sunday) from the real current date every load — see "This week is always live" below. |
| `recipes.json` | The shared recipe library: `{id, title, note, video, videoId}` per dish. `meal-dashboard.html`'s meal slots store only a recipe `id` (see Firestore schema below) and resolve title/note/video against this file at render time — single source of truth, no duplicated copies per slot. |
| `firestore-rules.md` | Firestore security rules for `mealPlans`/`mealThreads` (open read/write, scoped to just those two collections) — **designed, not wired yet**, see Setup below. |
| `README.md` | User-facing docs (English) for a non-technical maintainer — how to add a recipe / a hub card, how to deploy. |
| `project-knowledge.md` | This file. |

## This week is always live — never hardcode dates again

`meal-dashboard.html` computes the current Monday–Sunday week from `new Date()`
at load (`todayDate`/`todayIndex`/`mondayDate`/`weekDates`/`dayNums` near the top
of the `<script>`), and derives `weekId` (Monday's ISO date, e.g. `"2026-09-08"`)
— this is the key the future Firestore docs will be keyed by (see below). **Do
not go back to a hardcoded `dayNums` array or `todayIndex` constant** — that was
the mockup-era shortcut this was built to replace, per an explicit user ask.

## Architecture: what's static JSON vs. what's (planned) Firestore

Two kinds of data, deliberately split the same way `wg-groupbuy` splits static
`data-<date>.json` files from live `paidStatus`/`adjustments` Firestore
collections:

- **Static, repo-committed JSON** (`recipes.json`, `hub-cards.json`) — changes
  rarely (someone teaches the helper a new dish, or a new Home Hub section gets
  added), edited by hand and re-uploaded to GitHub. Fetched at boot; a missing/
  failed fetch should degrade gracefully (see `.catch()` on both fetches),
  not break the page.
- **Live, cross-device state** — staple picks, which recipe is assigned to each
  meal slot, status (planned/cooking/missing/done), and the message threads.
  This changes constantly through the week and needs to sync across every
  family member's device in real time — the reason for Firestore. **Not wired
  up yet** — there's no Firebase project for this app. Today it's plain
  in-memory JS state (`mealData`), reset on every page reload, but already
  **shaped exactly like the future Firestore documents** (see schema below) so
  wiring it in later is a mechanical swap of local mutation → `onSnapshot`/
  `setDoc`, not a redesign.

### Firestore schema (designed, not yet wired)

- `mealPlans/{weekId}` — one doc per week, `weekId` = that week's Monday ISO
  date (already computed in-page as the `weekId` constant). Fields: a map
  keyed by **date** (`"YYYY-MM-DD"`, one of that week's 7 dates), each holding
  a map per meal slot (`breakfast`, `breakfast_kids`, `lunch`, `lunch_kids`,
  `dinner`, `dinner_kids`) of `{ staple?, dishIds: [{id, videoPlaying}],
  status }`. Kids slots omit `staple` (no staple/rice selector for kids cards,
  by design — see Design conventions). `dishIds` holds recipe **references**
  (`id` into `recipes.json`), not embedded copies — matches the in-memory
  shape already used by `mealPlansByDate` today. **A date only appears in the
  map once something has actually been saved against it** — an untouched day
  has no key at all, not an empty placeholder; the dashboard renders a
  missing date as a fully empty day via `peekDayData()`. This date layer was
  added after an earlier version kept one flat set of 6 slots for the whole
  week — the day-strip tabs looked functional but silently showed identical
  data on every day (see Troubleshooting).
- `mealThreads/{weekId}` — one doc per week, same date-then-slot nesting as
  `mealPlans`, map per slot of message arrays (`{from, text_zh, text_tl,
  unread?}`) — mirrors groupbuy's one-doc-per-round pattern for `adjustments`
  (plenty of headroom at this household's scale).

`firestore-rules.md` already matches these two collection names — keep them in
sync if either changes.

### Setup — wiring Firestore in for real (do this in a future session, once a Firebase project exists)

1. **New Firebase project** (or ask whether to reuse the `wg-groupbuy` one —
   don't assume either way, confirm with the user first). console.firebase.google.com
   → create project → enable Firestore Database.
2. Paste the project's `firebaseConfig` into `meal-dashboard.html` (apiKey,
   authDomain, projectId, storageBucket, messagingSenderId, appId) — these
   aren't secrets, access control comes from the security rules, not from
   hiding the config.
3. Apply `firestore-rules.md` in Firebase console → Firestore Database → Rules.
4. Swap `mealData`'s local mutations (`mealData[slotKey].status = ...`, etc.)
   for `setDoc`/`updateDoc` calls against `mealPlans/{weekId}`, and subscribe
   with `onSnapshot` instead of reading the in-memory object directly — the
   shape is already correct, this is the mechanical part.
5. Update the "unread" badge plumbing (see below) to read the live count from
   `mealThreads` instead of the `localStorage` stand-in.
6. Confirm cross-device sync by opening the dashboard in two tabs/devices and
   checking a change in one shows up in the other.

## Cross-page conventions already established

- **Theme** (`localStorage['wg-theme']`, values `light`/`dark`/`auto`) and
  **language** (`localStorage['wg-lang']`, values `en`/`zh`/`tl`) are shared
  across `index.html` and `meal-dashboard.html` via the same `localStorage`
  keys — picking either on one page carries to the other. Any new page added
  to this app should read/write the same two keys, not invent its own.
- **Unread-messages badge**: `meal-dashboard.html` writes its live total
  unread-helper-message count to `localStorage['wg-meals-unread']` every time
  the bell updates (see `renderBell()`). `index.html` reads that key to show a
  red app-icon-style badge on the top-left corner of the Meals card — this is
  a **`localStorage` stand-in**, not a real shared backend; it only reflects
  reality if the meal dashboard has been opened in that same browser at some
  point. If `index.html` is opened standalone/fresh, it falls back to a seeded
  default of `2` purely so the badge concept is visible in a demo — replace
  this fallback logic once the badge is Firestore-driven for real.
- **Notification badge is on the card itself** (top-left, overlapping the
  corner like an iOS app icon), not a bell icon on the Home Hub — that was an
  explicit design choice. The badge lives in a `.hub-card-wrap` div *without*
  `overflow:hidden` (the inner `.hub-card` keeps `overflow:hidden` for its tab
  corner styling) — putting the badge inside the clipped element cuts it off
  at the card edge, learned the hard way; keep this wrapper structure for any
  future card that needs a badge.

## Design conventions already established (don't relitigate unless asked)

- Warm parchment aesthetic: dotted background, Fraunces serif headers, Karla
  sans body, dashed dividers — same visual language as `wg-groupbuy`'s
  receipt-style dashboard, adapted to a softer household-app palette (see
  `:root` CSS vars in either page for the full light/dark token set).
- **Light/Dark/Auto is a segmented pill control** (`.theme-switch`/`.theme-seg`),
  not a single cycling icon button — this was an explicit revision after an
  icon-only version was tried first; match the screenshot-driven pill design if
  rebuilding it anywhere.
- Kids cards: dashed border + a dedicated plum/lavender accent (`--kids-accent`)
  used consistently on all three Kids tabs and borders, distinct from the
  per-meal mustard/teal/brick used on the Adults row. Kids cards have **no
  staple/rice selector** (explicit ask) and their tab always reads
  "🧒 {Meal} · Kids" vs. the adult tab's "{Meal} · Adults" — both audiences are
  labeled explicitly, not just the Kids one, per an explicit "make it obvious
  which is for which" ask.
- Message input has **no Send button** — Enter/Return sends, with a small
  italic "press Enter to send" hint under the field instead (explicit
  revision: the button was overlapping the on-screen keyboard on mobile).
  The input is refocused after sending since `renderMeals()` rebuilds the
  whole card and would otherwise drop focus mid-conversation.
- Chat-style message thread: helper (received) messages are left-aligned, your
  (sent) messages are right-aligned, both capped at `max-width:85%` so the
  alignment is visible even when a line wraps to two lines. **The original
  message is never boxed; only its translation gets a colored box** (mustard
  for the helper's translation, teal for yours) — this was flipped from an
  earlier version that boxed the whole bubble, per explicit request, to keep
  visual focus on the translation. A message's displayed language is
  **fixed by who sent it** (helper's original is Tagalog + Chinese
  translation; yours is Chinese original + Tagalog translation) — this must
  **never** depend on the current EN/中文/TL UI toggle, which only affects
  interface copy, not conversation history. This bug already happened once
  (language toggle was flipping which language showed as "original") — don't
  reintroduce it.
- i18n pattern: a plain per-page `i18n` object (`{en:{...}, zh:{...}, tl:{...}}`)
  plus a `t(key)` lookup that falls back to returning the key itself if a
  language is missing that key — this is *intentional*, not a bug: it's why
  `days`' `Mon`/`Tue`/etc. keys only need explicit `zh` entries (Chinese day
  names genuinely differ) while `en`/`tl` fall through to the literal
  `Mon`/`Tue`/etc. strings unchanged — **don't add redundant identity entries
  for a language that should just fall through.**
- Any element with `data-i18n`/`data-i18n-ph` gets updated by a `syncLangUI()`
  sweep on both language switch *and* page boot (needed once language became
  `localStorage`-persisted — a value other than the HTML's hardcoded default
  needs that sweep to run before the user ever clicks anything).

## Troubleshooting / lessons already learned

- **The day-strip used to be purely cosmetic.** The first version of the
  in-memory meal state was a single flat set of 6 meal slots for the whole
  week, so clicking Mon/Tue/Wed/... re-rendered the exact same dishes/status/
  thread every time — nothing was actually keyed by date. Fixed by nesting
  state under `mealPlansByDate[dateIso][slotKey]` (see Firestore schema
  above): an untouched date has no entry and renders empty via
  `peekDayData()`; only `getOrCreateDayData()` (called from an actual add/
  edit/send) persists a date, which also drives the small dot shown on a
  day-tab that has data (`dayHasData()`). Today's date is pre-seeded with
  demo content so the app opens with something planned; every other day
  starts genuinely empty. Don't flatten this back to slot-only keys.
- **YouTube embeds require an `http://`/`https://` origin.** Opening any of
  these pages via `file://` (double-click) makes YouTube's iframe player throw
  "Error 153 / Video player configuration error" for **every** video,
  regardless of that video's actual embed permissions — this cost real
  debugging time before the cause was found (even the famously-always-public
  `dQw4w9WgXcQ` failed under `file://`). **Always test via a local server**:
  `python3 -m http.server` from this folder, then open `http://localhost:PORT/`.
  Now that `fetch()` calls were added for the JSON files, this matters even
  more — Chrome in particular also blocks local `fetch()` under `file://`
  (CORS), so JSON loading may silently fail too under a plain double-click.
- **Video iframes must only get `autoplay=1` on the single render right after
  the click**, via the one-shot `autoplayDish` flag (set on click, cleared at
  the end of the very next `renderMeals()`). Because `renderMeals()` rebuilds
  every card's HTML from scratch on *any* state change (status toggle, staple
  pick, language switch, sending a message...), a permanently-set
  `autoplay=1` in a dish's stored data would restart every already-playing
  video on every unrelated re-render. Don't move `autoplay=1` back into the
  dish's persisted state.
- **"Add a new dish" via link accepts either a plain YouTube URL or a pasted
  `<iframe>` embed snippet** — `extractYouTubeId()` checks for an
  `<iframe ... src="...">` first and pulls the ID from that if present, else
  scans the raw pasted text. Keep both paths working if this function is
  touched again.
- A missing/renamed recipe `id` referenced by a meal slot shows a visible
  "⚠️ {id} — Recipe not found" line instead of throwing (see the `findRecipe`
  guard in `meal-dashboard.html`'s dish renderer) — same defensive philosophy
  as `wg-groupbuy`'s missing-product-key handling. Any future code path that
  reads a dish by `id` needs the same guard, not a bare `.find(...).title`.
