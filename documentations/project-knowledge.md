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
| `index.html` | Home Hub landing page. Fetches `hub-cards.json` and renders one real card per entry — no placeholder/ghost card anymore (removed per explicit ask). If the fetch fails or returns nothing usable, falls back to a hardcoded Meals-only card (`FALLBACK_HUB_CARDS`) so the hub is never blank — the one deliberate exception to "never hardcode a card's data," kept in sync with `hub-cards.json`'s meals entry. |
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
  is now a real, working config (see Setup below).

### Firestore schema (current — dynamic cards / unified chat)

- `mealPlans/{weekId}` — one doc per week, `weekId` = that week's Monday ISO
  date. Field: `cards`, an array of `{id, audience: 'adult'|'kids', mealType,
  staple?, dishIds: [{id}], status}`. Only meals someone actually added exist
  — there is no fixed set of slots. `staple` only applies to adult cards.
  `dishIds` holds recipe **references** (`id` into the recipe library), not
  embedded copies. **`videoPlaying` is deliberately NOT in this schema** —
  it's transient per-viewer UI state, kept in a page-local
  `videoPlayingState` map keyed `"cardId::dishIndex"`, never written to
  Firestore (persisting it would restart every viewer's video player on any
  unrelated change to the doc). The plan is week-level, not per-day — the
  day strip is contextual/navigational only, same as the original design.
- `mealThreads/{weekId}` — one doc per week. Field: `messages`, a single
  array for the whole week (not split per meal slot). Each message:
  `{from, text, origLang, translations:{lang:text}, unread?, senderId?,
  senderName?, ts?}`. `senderName`/`senderId` are only set on `from:"you"`
  messages, captured from `localStorage['wg-username']`/`['wg-userid']` at
  send time (a later rename doesn't rewrite chat history).
- `familyMembers/{memberId}` — one doc per device, `memberId` = a random ID
  generated client-side (`crypto.randomUUID()`) and kept in
  `localStorage['wg-userid']`. Fields: `{ name, updatedAt }`.
- `recipeLibrary/main` — one shared doc (field `recipes`, same shape as
  `recipes.json`) — the live, Firestore-backed recipe library. Seeded once
  from `recipes.json` the first time this doc is created; every add/edit
  from the UI writes the whole array back here. `recipes.json` remains the
  fallback when Firebase isn't configured and the seed data for a
  from-scratch install.

`firestore-rules.md` matches all four collection names — keep them in sync
if any change.

**One-time migration**: `initFirestoreSync()` in `meal-dashboard.html`
detects the old fixed-slot schema (`cards` missing on the plan doc /
`messages` missing on the thread doc) and migrates automatically, once —
see `migrateOldMealPlan()`/`migrateOldThread()`. Old fields are left in the
Firestore doc untouched (harmless, just unread by the new code) rather than
deleted.

### Name entry (built)

Both `index.html` and `meal-dashboard.html` show a first-visit modal asking
"What's your name?" if `localStorage['wg-username']` is unset (skippable).
Saving writes to `localStorage` immediately and, if Firestore is configured,
upserts `familyMembers/{wg-userid}`. The Home Hub shows a small "Hi, {name}"
button (tap to reopen the modal and change it) in the header. The saved name
is what shows instead of the generic "You" label on a sent chat message in
`meal-dashboard.html` — see `senderName` above. If a user reaches the meal
dashboard directly (bookmarked, skipping the hub) with no name set yet, the
same modal appears there too, and sending a message before naming yourself
prompts for it first.

### Setup — Firebase project config: DONE

The `wg-family-assistant` Firebase project's config is pasted into both
`index.html` and `meal-dashboard.html` (identical in both, as required —
they must point at the same Firestore project). `getAnalytics` was
deliberately left out — this app doesn't use Firebase Analytics, only
Firestore, so pulling in that extra SDK would be dead weight.

**Still needed before this actually syncs (if not already done):**
1. Confirm Firestore Database is enabled for this project in the Firebase
   Console (Build → Firestore Database → Create database, if not already
   done).
2. Paste `firestore-rules.md`'s rules block into Firestore Database → Rules
   → publish. Without this, reads/writes will be rejected by the default
   rules.
3. Re-upload edited files to GitHub.
4. Confirm cross-device sync: open the dashboard on two devices/tabs, change
   a status or send a message on one, watch it appear on the other. Also
   confirm the Home Hub's badge count updates live, and that a name entered
   on one device shows up correctly labeled on another.

## Cross-page conventions already established

- **Language** (`localStorage['wg-lang']`, values `en`/`zh`, default `zh`)
  is shared across `index.html` and `meal-dashboard.html` via the same
  `localStorage` key — picking either on one page carries to the other.
  Any new page added to this app should read/write the same key, not
  invent its own. **There is no theme toggle** — light/dark/auto mode was
  removed entirely (2026-09-17); both pages render a single fixed light
  palette. Don't reintroduce a `data-theme`/`wg-theme` mechanism unless
  explicitly asked again.
- **Unread-messages badge**: `index.html` subscribes directly to this week's
  `mealThreads/{weekId}` doc (`subscribeMealsBadge()`) and counts unread
  messages not sent by this device (`!isMine(m)`) across the unified
  `messages` array. This means the badge is live across devices, not just
  within one browser, and needs no page to have been opened first. If
  Firestore isn't configured, the badge simply doesn't show.
- **Notification badge is on the card itself** (top-left, overlapping the
  corner like an iOS app icon), not a bell icon on the Home Hub — that was an
  explicit design choice. The badge lives in a `.hub-card-wrap` div *without*
  `overflow:hidden` (the inner `.hub-card` keeps `overflow:hidden` for its tab
  corner styling) — putting the badge inside the clipped element cuts it off
  at the card edge, learned the hard way; keep this wrapper structure for any
  future card that needs a badge.

## Design conventions already established (don't relitigate unless asked)

- **WeChat-style palette (2026-09-17):** white/near-white background
  (`--bg:#F7F7F7`, `--paper:#FFFFFF`) with an **orange sticky top banner**
  (`--banner-bg:#F2711C`, white banner text/icons). Card accents (links,
  active toggle states, chip highlights) reuse the same orange family via
  the `--teal` CSS variable (kept that name for continuity with earlier
  code, even though the color itself is now orange, not teal — don't be
  confused by the variable name if this file is touched again). This
  replaced the earlier dark-teal-banner / parchment-paper aesthetic.
- System font stack, no Google Fonts: `--font-body` (system sans,
  `-apple-system`/`PingFang SC`/`Microsoft YaHei` etc.) and `--font-head`
  (system serif, `Songti SC`/`STSong`/`SimSun`) — dropped Fraunces/Karla
  since Google Fonts is unreliable behind China's firewall, which defeats
  the point of a WeChat-optimized site.
- **No theme toggle.** Removed entirely (2026-09-17) — see above.
- Kids cards: dashed border + a dedicated plum/lavender accent (`--kids-accent`)
  used consistently on all Kids tabs and borders, distinct from the
  per-meal mustard/orange/brick used on the Adults row. Kids cards have **no
  staple/rice selector** (explicit ask) and their tab always reads
  "🧒 {Meal} · Kids" vs. the adult tab's "{Meal} · Adults" — both audiences are
  labeled explicitly, not just the Kids one, per an explicit "make it obvious
  which is for which" ask.
- Chat-style message thread: helper (received) messages are left-aligned, your
  (sent) messages are right-aligned, both capped at `max-width:85%` so the
  alignment is visible even when a line wraps to two lines. **The original
  message is never boxed; only its translation gets a colored box** — this
  keeps visual focus on the translation. A message's displayed original
  language is **fixed by who sent it and what they typed**
  (`detectLangHeuristic()`), never by the current EN/中文 UI toggle, which
  only affects interface copy, not conversation history.
- i18n pattern: a plain per-page `i18n` object (`{en:{...}, zh:{...}}`) plus
  a `t(key)` lookup that falls back to returning the key itself if a
  language is missing that key. Tagalog (`tl`) was removed from both
  pages' UI-level `i18n` dictionaries (2026-09-17, see below) — **but is
  still used in the chat's language-detection/translation logic**
  (`TAGALOG_MARKERS`, `detectLangHeuristic()`), since the helper can still
  type Tagalog messages; only the interface chrome dropped it.
- Any element with `data-i18n`/`data-i18n-ph`/`data-i18n-title` gets updated
  by a `syncLangUI()` sweep on both language switch *and* page boot (needed
  since language is `localStorage`-persisted — a value other than the
  HTML's hardcoded default needs that sweep to run before the user ever
  clicks anything).

## Chat label / recipe editing / caching / null-guard patterns (carried forward)

- **Chat label is per-viewer, not per-sender.** A message you send always
  shows "You" on your own device; the same message shows your actual name
  on anyone else's device, via `senderId` compared against the viewing
  device's own `getUserId()` (`isOwnMessage()`/`chatLabelFor()`). Messages
  with no `senderId` (pre-dating this field) fall back to "You" for
  everyone.
- **Chat bubble alignment is per-viewer too**, via the same shared
  `isOwnMessage(m)` helper driving the `.mine` class — not keyed off
  `m.from` directly, which only drives the helper/you pairing and
  translation-box styling.
- **Stale-cache handling**: both pages send `Cache-Control: no-cache` /
  `Pragma: no-cache` / `Expires: 0` meta tags, append a `?v=Date.now()`
  cache-busting query string to `hub-cards.json`/`recipes.json` fetches,
  and force a real reload on `pageshow` when `event.persisted` is true.
- **Null/blank hardening**: incoming `mealPlans` cards are run through
  `sanitizeCard()`; incoming `mealThreads` messages are filtered to ones
  with actual content; `recipes.json`/`hub-cards.json`/`recipeLibrary`
  entries missing required fields are dropped at fetch/sync time rather
  than reaching the renderer.
- A missing/renamed recipe `id` referenced by a card shows a visible
  "⚠️ {id} — Recipe not found" line instead of throwing (`findRecipe`
  guard in the dish renderer).
- **YouTube embeds require an `http://`/`https://` origin** — `file://`
  breaks both the iframe player (Error 153) and `fetch()` (CORS). Always
  test via a local server (`python3 -m http.server`), not double-clicking
  the file.
- **Video iframes only get `autoplay=1` on the single render right after
  the click**, via the one-shot `autoplayDish` flag (set on click, cleared
  at the end of the very next `renderMeals()`) — don't move `autoplay=1`
  into the dish's persisted state, or every already-playing video restarts
  on any unrelated re-render.
- **"Add a new dish" via link accepts either a plain YouTube URL or a
  pasted `<iframe>` embed snippet** — `extractYouTubeId()` checks for an
  `<iframe ... src="...">` first, else scans the raw pasted text.

## Major redesign — WeChat-optimized rebuild (2026-09-17)

Both `index.html` and `meal-dashboard.html` were rebuilt for WeChat's in-app
browser and a Mandarin-first household. **This was a breaking schema
change.**

### What changed and why
- **Tailwind CDN** (`https://cdn.tailwindcss.com`) added to both pages,
  alongside the existing CSS-custom-property color system. Static shell
  markup (header, modals) leans on Tailwind utility classes; the
  dynamically-rendered meal cards, dishes, and chat bubbles keep their
  original hand-written CSS classes (`.card`, `.dish`, `.msg`, etc.).
- **Google Fonts removed entirely.**
- **Tagalog removed from the UI language toggle** — see Design conventions.
- **Sticky top banner** on both pages, holds the page title and language
  pills (theme toggle since removed — see below).
- **Meal cards are no longer 6 fixed always-shown slots** — dynamic `cards`
  array, "+ Add a meal" ghost card expands an inline draft card (pick
  audience → meal type → recipe) instead of a modal.
- **Chat is now ONE unified thread for the whole week**, not per-slot
  mini-threads — `mealThreads/{weekId}` is `{ messages: [...] }`.
- **Migration on first load**: `migrateOldMealPlan()`/`migrateOldThread()`
  detect and convert the old schema automatically, once.

### Not done in this pass (still true)
- `README.md` was not rewritten for the new flow — it still describes the
  old fixed-6-slot / per-meal-thread behavior. Needs a pass before handing
  this off to a non-technical user.
- No live-browser testing has been performed in any session so far — only
  `node --check` (JS syntax), HTML tag-balance checks, and JSON validation.
  Test via a local server before trusting this in production.
- The Tailwind CDN script is itself an external request — if WeChat's
  in-app browser in mainland China ever has trouble reaching it, the
  fallback is self-hosting Tailwind's compiled CSS instead of the Play CDN
  script.

## Follow-up fixes to the WeChat redesign (2026-09-17)

1. **Chat moved back into the Meals view, as a card — no separate tab.**
   The tab switcher from the first redesign pass is gone. Chat is a
   bordered card (`.chat-card`) between the day strip and the meal cards,
   with its own scrollable message list and a 🗑️ clear button in its
   header. The sticky bottom input bar is always visible now. Unread
   messages are marked read automatically whenever `renderChat()` runs.
2. **Chat now translates into every language it isn't already written
   in**, not just English — a message's `translations` field is an object
   keyed by the languages it needs (`{en, zh, tl} minus origLang`), fetched
   in parallel via `Promise.all`.
3. **Recipes are now Firestore-backed, not session-only** — root cause of
   "the YouTube link doesn't save." A `recipeLibrary/main` Firestore
   document is now the source of truth, seeded from `recipes.json`, kept
   in sync live via `onSnapshot`; every add/edit calls
   `writeRecipeLibrary()`.
4. **Recipe titles are now bilingual** (`title: {en, zh}`), and the
   displayed name follows the site's language toggle. The add/edit-dish
   modal has separate Chinese/English name fields; leaving one blank
   copies the other into it. Recipe notes/instructions stay
   single-language (extra detail for the helper goes through chat
   instead). A plain string `title` (legacy data) is still handled.

## Theme removal + WeChat color pass (2026-09-17)

- **Light/Dark/Auto theme toggle removed entirely** from both pages — the
  ☀️/🌙/🌗 `.icon-btn[data-theme-choice]` buttons, the `applyTheme()`/
  `themePref`/`localStorage['wg-theme']` JS, and the
  `@media (prefers-color-scheme: dark)` / `:root[data-theme="dark"]` CSS
  blocks are all gone. Both pages now render a single fixed light palette.
  Don't reintroduce this unless explicitly asked again.
- **Palette switched to a WeChat-typical look**: white/near-white
  background (`--bg:#F7F7F7`, `--paper:#FFFFFF`) with an **orange sticky
  top banner** (`--banner-bg:#F2711C`, white banner text). See Design
  conventions above for the full color rationale, including the note that
  the `--teal` CSS variable name was kept for continuity even though its
  value is now orange.
- Files are now also sent to the user as real downloadable attachments
  (`SendUserFile`) alongside every `project_write`, not just saved into
  the claude.ai Project — a gap flagged directly by the user, since
  `project_write` alone never produced a file they could download.
