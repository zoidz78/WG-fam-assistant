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
  family member's device in real time — Firestore. **The code is wired up**
  (both pages import the Firebase modular SDK and call `onSnapshot`/`setDoc`/
  `updateDoc` against `mealPlans/{weekId}` and `mealThreads/{weekId}`), but
  the `firebaseConfig` object in both `index.html` and `meal-dashboard.html`
  is still a `"PASTE_ME"` placeholder — **a real Firebase project needs to be
  created and its config pasted into both files** before any of this actually
  syncs (see Setup below). Until then everything degrades gracefully to
  local-only, in-memory state for that single page load.

### Firestore schema (code wired, project not created yet)

- `mealPlans/{weekId}` — one doc per week, `weekId` = that week's Monday ISO
  date (computed in-page as the `weekId` constant, same formula duplicated in
  `index.html` as `currentWeekId()` for the badge). Fields: a map per meal
  slot (`breakfast`, `breakfast_kids`, `lunch`, `lunch_kids`, `dinner`,
  `dinner_kids`) of `{ staple?, dishIds: [{id}], status }`. Kids slots omit
  `staple` (no staple/rice selector for kids cards, by design — see Design
  conventions). `dishIds` holds recipe **references** (`id` into
  `recipes.json`), not embedded copies. **`videoPlaying` is deliberately NOT
  in this schema** — it's transient per-viewer UI state, kept in a
  page-local `videoPlayingState` map keyed `"slotKey::dishIndex"`, never
  written to Firestore (persisting it would restart every viewer's video
  player on any unrelated change to the doc).
- `mealThreads/{weekId}` — one doc per week, map per slot of message arrays
  (`{from, text_zh, text_tl, unread?, senderName?}`) — mirrors groupbuy's
  one-doc-per-round pattern for `adjustments`. `senderName` is only set on
  `from:"you"` messages, captured from `localStorage['wg-username']` **at
  send time** (a later rename doesn't rewrite chat history).
- `familyMembers/{memberId}` — one doc per device, `memberId` = a random ID
  generated client-side (`crypto.randomUUID()`) and kept in
  `localStorage['wg-userid']`. Fields: `{ name, updatedAt }`. Written by the
  name-entry modal (see below); not yet read anywhere else, but it's there so
  a future page (e.g. a family-member picker) doesn't need a schema change.

`firestore-rules.md` matches all three collection names — keep them in sync
if any change.

### Name entry (built)

Both `index.html` and `meal-dashboard.html` show a first-visit modal asking
"What's your name?" if `localStorage['wg-username']` is unset (skippable).
Saving writes to `localStorage` immediately and, if Firestore is configured,
upserts `familyMembers/{wg-userid}`. The Home Hub shows a small "Hi, {name}"
button (tap to reopen the modal and change it) next to the theme switch.
The saved name is what now shows instead of the generic "You" label on a
sent chat message in `meal-dashboard.html` — see `senderName` above. If a
user reaches the meal dashboard directly (bookmarked, skipping the hub) with
no name set yet, the same modal appears there too, and sending a message
before naming yourself prompts for it first.

### Setup — Firebase project config: DONE

The `wg-family-assistant` Firebase project's config is now pasted into both
`index.html` and `meal-dashboard.html` (identical in both, as required —
they must point at the same Firestore project). `getAnalytics` was
deliberately left out — this app doesn't use Firebase Analytics, only
Firestore, so pulling in that extra SDK would be dead weight.

**Still needed before this actually syncs:**
1. Confirm Firestore Database is enabled for this project in the Firebase
   Console (Build → Firestore Database → Create database, if not already
   done).
2. Paste `firestore-rules.md`'s rules block into Firestore Database → Rules
   → publish. Without this, reads/writes will be rejected by the default
   rules.
3. Re-upload both edited HTML files to GitHub.
4. Confirm cross-device sync: open the dashboard on two devices/tabs, change
   a status or send a message on one, watch it appear on the other. Also
   confirm the Home Hub's badge count updates live, and that a name entered
   on one device shows up correctly labeled on another.

## Cross-page conventions already established

- **Theme** (`localStorage['wg-theme']`, values `light`/`dark`/`auto`) and
  **language** (`localStorage['wg-lang']`, values `en`/`zh`/`tl`) are shared
  across `index.html` and `meal-dashboard.html` via the same `localStorage`
  keys — picking either on one page carries to the other. Any new page added
  to this app should read/write the same two keys, not invent its own.
- **Unread-messages badge**: `index.html` subscribes directly to this week's
  `mealThreads/{weekId}` doc (`subscribeMealsBadge()`) and counts unread
  helper messages across all slots itself — no more `localStorage`
  stand-in. This means the badge is live across devices, not just within one
  browser, and needs no page to have been opened first. If Firestore isn't
  configured yet (placeholder `firebaseConfig`), the badge simply doesn't
  show — no seeded fallback count anymore.
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

## Latest round of fixes (chat labels, recipe editing, caching, null-guards)

- **Chat label is per-viewer, not per-sender.** A message you send always
  shows "You" on your own device; the same message shows your actual name
  on anyone else's device. This needed a `senderId` (the sending device's
  `wg-userid`) stored on every `from:"you"` message alongside `senderName`;
  rendering compares `senderId` against the *viewing* device's own
  `getUserId()` (see `chatLabelFor()`) rather than trusting a fixed label.
  Messages saved before this field existed have no `senderId` and fall back
  to "You" for everyone (previous behavior), not a guess.
- **Recipes can now be edited from the UI**, not just added. A ✏️ button
  sits next to a dish's ✕ remove button on the meal card, and next to each
  entry in the "choose a dish" list (`openEditDishModal()`) — both open the
  same add-dish modal pre-filled with that recipe's current title/note/video,
  now titled "Edit dish". Saving mutates the existing `dishLibrary` entry in
  place (same object every card already points to via `findRecipe()`), so a
  dish already sitting in a meal slot updates immediately. A recipe with a
  blank note shows "No note yet — tap ✏️ to add one" instead of empty space,
  so incomplete entries (e.g. `hainan_chicken_rice`'s missing `videoId`) are
  visibly flagged rather than silently blank. Like "add a new dish", this
  only lasts the browser session unless also hand-edited into `recipes.json`
  — no Firestore backing for the recipe library itself, by design (see
  Architecture above).
- **Stale-cache fix, same as wg-groupbuy**: both pages now send
  `Cache-Control: no-cache` / `Pragma: no-cache` / `Expires: 0` meta tags,
  append a `?v=Date.now()` cache-busting query string to the `hub-cards.json`
  and `recipes.json` fetches (GitHub Pages' CDN can otherwise serve a stale
  copy for a while after a re-upload), and force a real reload on
  `pageshow` when `event.persisted` is true (mobile back-forward-cache
  restoring a stale page after navigating between the hub and the
  dashboard and back).
- **Null/blank hardening, same defensive pattern as wg-groupbuy's
  missing-product-key guard**: an incoming `mealPlans` doc is run through
  `sanitizeSlot()` (drops dish entries with no usable `id`, falls back to a
  valid `status`/`staple` if the stored value is missing or garbage) instead
  of being trusted as-is; incoming `mealThreads` messages are filtered to
  ones that actually have a `from` and some text; `recipes.json` and
  `hub-cards.json` entries missing their required fields are dropped at
  fetch time rather than reaching the renderer. **If there were other
  specific null/blank bugs found in the wg-groupbuy project beyond this
  pattern, they haven't been cross-checked here yet — flag the specific
  cases in a future session so they can be verified against this codebase
  too.**

## Troubleshooting / lessons already learned

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
