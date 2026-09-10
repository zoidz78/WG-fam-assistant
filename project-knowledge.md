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
| `recipes.json` | **One-time seed only** for the recipe library — the live, editable copy now lives in Firestore (`recipeLibrary/library`, see Firestore schema below) and syncs across devices. This file is only read the very first time that Firestore doc doesn't exist yet, or as a same-tab fallback if Firebase isn't configured. `{id, title, note, video, videoId}` per dish; meal slots store only a recipe `id`, resolved against the live library at render time — single source of truth, no duplicated copies per slot. |
| `tagalog-markers.json` | `{words: [...], phrases: [...]}` — the word/phrase list `detectLangHeuristic()` checks a message against to guess Tagalog vs. English (see Messages & translation below). Fetched at boot with the same cache-busting pattern as `recipes.json`; add a word or phrase here to fix a message that isn't getting detected/translated correctly, no code change needed. |
| `firestore-rules.md` | Firestore security rules for `mealPlans`/`mealThreads`/`familyMembers` (open read/write) — **live**, already applied in the Firebase project this app uses. |
| `README.md` | User-facing docs (English) for a non-technical maintainer — how to add a recipe / a hub card, how to deploy. |
| `project-knowledge.md` | This file. |

## This week is always live — never hardcode dates again

`meal-dashboard.html` computes the current Monday–Sunday week from `new Date()`
at load (`todayDate`/`todayIndex`/`mondayDate`/`weekDates`/`dayNums` near the top
of the `<script>`), and derives `weekId` (Monday's ISO date, e.g. `"2026-09-08"`)
— this is the key the future Firestore docs will be keyed by (see below). **Do
not go back to a hardcoded `dayNums` array or `todayIndex` constant** — that was
the mockup-era shortcut this was built to replace, per an explicit user ask.

## Architecture: what's static JSON vs. what's Firestore

Two kinds of data, deliberately split the same way `wg-groupbuy` splits static
`data-<date>.json` files from live `paidStatus`/`adjustments` Firestore
collections:

- **Static, repo-committed JSON** (`hub-cards.json`, `tagalog-markers.json`)
  — changes rarely (a new Home Hub section gets added, or a real message
  turns out to need a new marker word/phrase), edited by hand and
  re-uploaded to GitHub. Fetched at boot; a missing/failed fetch should
  degrade gracefully (see `.catch()` on each fetch — `tagalog-markers.json`
  falls back to a small hardcoded word/phrase list baked into
  `meal-dashboard.html` itself), not break the page. `recipes.json` used to
  live in this bucket too, but **no longer does** — see the recipe library
  note under Live, cross-device state below; it's now only a one-time seed
  file, not something you'd routinely hand-edit and re-upload.
- **Live, cross-device state** — staple picks, which recipe is assigned to each
  meal slot, status (planned/cooking/missing/done), the message threads, and
  (as of the latest change) **the recipe library itself** — title, note, and
  video link per dish. All of this changes and needs to sync across every
  family member's device in real time — Firestore. **Wired up and live**
  (both pages import the Firebase modular SDK and call `onSnapshot`/`setDoc`/
  `updateDoc` against `mealPlans/{weekId}` and `mealThreads/{weekId}`) — the
  `firebaseConfig` object in both `index.html` and `meal-dashboard.html` has
  the real `wg-family-assistant` project's values pasted in (identical in
  both, as required — see Setup below).

### Firestore schema (wired and live)

- `mealPlans/{weekId}` — one doc per week, `weekId` = that week's Monday ISO
  date (computed in-page as the `weekId` constant, same formula duplicated in
  `index.html` as `currentWeekId()` for the badge). Fields: a map keyed by
  **date** (`"YYYY-MM-DD"`, one of that week's 7 dates), each holding a map
  per meal slot (`breakfast`, `breakfast_kids`, `lunch`, `lunch_kids`,
  `dinner`, `dinner_kids`) of `{ staple?, dishIds: [{id}], status }`. Kids
  slots omit `staple` (no staple/rice selector for kids cards, by design —
  see Design conventions). `dishIds` holds recipe **references** (`id` into
  `recipes.json`), not embedded copies. **`videoPlaying` is deliberately NOT
  in this schema** — it's transient per-viewer UI state, kept in a
  page-local `videoPlayingState` map keyed `"dateIso::slotKey::dishIndex"`,
  never written to Firestore (persisting it would restart every viewer's
  video player on any unrelated change to the doc). **A date only appears in
  the map once something has actually been saved against it** — an untouched
  day has no key at all; `peekDayPlan()`/`peekDayThreads()` render a missing
  date as a fully empty day. Writes use a Firestore dot-path field update
  (e.g. `updateDoc(mealPlanRef, {"2026-09-10.lunch": newSlotData})`), which
  creates the nested date map automatically the first time — no need to
  pre-seed every date. This date layer replaced an earlier flat, slot-only
  shape where the day-strip tabs looked functional but silently showed
  identical data on every day (see Troubleshooting). The testing-only
  "Reset this week" button (see below) wipes this whole doc back to `{}`
  too, alongside `mealThreads/{weekId}`.
- `mealThreads/{weekId}` — one doc per week, same date-then-slot nesting as
  `mealPlans`, map per slot of message arrays (`{from, text_zh, text_tl,
  readBy?, senderName?}`) — mirrors groupbuy's one-doc-per-round pattern for
  `adjustments`. `senderName` is only set on `from:"you"` messages, captured
  from `localStorage['wg-username']` **at send time** (a later rename
  doesn't rewrite chat history). `readBy` is an array of viewer ids
  (`getUserId()`'s `localStorage['wg-userid']`) accumulated onto a message
  as each person actually reads it — **not** a single shared "unread"
  boolean; see Troubleshooting for why that was a real bug. The testing-only
  "Reset this week" button (see below) wipes this whole doc back to `{}` —
  every date, not just the one being viewed — along with `mealPlans/{weekId}`
  (see that entry above).
- `familyMembers/{memberId}` — one doc per device, `memberId` = a random ID
  generated client-side (`crypto.randomUUID()`) and kept in
  `localStorage['wg-userid']`. Fields: `{ name, updatedAt }`. Written by the
  name-entry modal (see below); not yet read anywhere else, but it's there so
  a future page (e.g. a family-member picker) doesn't need a schema change.
- `recipeLibrary/library` — a **single fixed doc** (not per-week like the
  other two), one top-level field per dish keyed by recipe id:
  `{ [dishId]: { title, note, video, videoId } }`. Seeded once from
  `recipes.json` the first time this doc doesn't exist (`initRecipeLibrarySync()`
  in `meal-dashboard.html`); after that, `recipes.json` is never read from
  again this session — Firestore is the live source of truth. Editing or
  adding a dish (`writeDish()`) writes a dot-path update using the dish id
  as the field name (e.g. `"century_congee.videoId"` — no literal dots
  allowed in a dish id for this reason, which snake_case ids already
  satisfy), so two people editing *different* dishes at the same time never
  clobber each other. A dish with no usable `title` is dropped by
  `sanitizeDish()` rather than rendering a blank card, same defensive
  philosophy as `sanitizeSlot()`.

`firestore-rules.md` matches all four collection names — keep them in sync
if any change.

### Name entry (built)

Both `index.html` and `meal-dashboard.html` show a first-visit modal asking
"What's your name?" if `localStorage['wg-username']` is unset (skippable).
Saving writes to `localStorage` immediately and, if Firestore is configured,
upserts `familyMembers/{wg-userid}`. **Both pages** show a small "Hi, {name}"
button (tap to reopen the modal and change it) next to the theme switch —
originally only the Home Hub had this, replicated onto `meal-dashboard.html`
per explicit ask ("any and all cards"). **Any future page added to this app
(chores, groceries, etc.) needs the same greeting button**, not just the
name-entry modal — copy `index.html`'s `.greeting`/`.greeting b` CSS, the
`<button class="greeting" id="greetingBtn" style="display:none;">` markup
(placed first inside the header's controls row), the `greetingHi` i18n key
in all three languages, and `renderGreeting()` (called at boot, after
saving a name, and after a language switch — it's language-dependent text).
The saved name is what now shows instead of the generic "You" label on a
sent chat message in `meal-dashboard.html` — see `senderName` above. If a
user reaches the meal dashboard directly (bookmarked, skipping the hub) with
no name set yet, the same modal appears there too, and sending a message
before naming yourself prompts for it first.

### Setup — Firebase project config: DONE

The `wg-family-assistant` Firebase project's config is pasted into both
`index.html` and `meal-dashboard.html` (identical in both, as required —
they must point at the same Firestore project). `getAnalytics` was
deliberately left out — this app doesn't use Firebase Analytics, only
Firestore, so pulling in that extra SDK would be dead weight. Firestore
Database is enabled and `firestore-rules.md`'s rules block is published in
the Firebase console, and the site is live and syncing across devices.

**Still needed right now:**
1. The `mealPlans`/`mealThreads` schema changed to nest by date (see
   Firestore schema above) — existing data written under the old flat
   (slot-only) shape won't be read by the new date-aware code; if there's
   anything in the live `mealPlans`/`mealThreads` docs worth keeping,
   migrate it by hand in the Firebase console before re-uploading,
   otherwise it'll just look like every day is empty (which, functionally,
   matches how an untouched day already renders).
2. A new `recipeLibrary` collection was added — re-publish
   `firestore-rules.md`'s updated rules block in Firebase console →
   Firestore Database → Rules, or writes to it will be rejected by the
   default deny-all rules.
3. Re-upload `meal-dashboard.html` (and `index.html`, if it changed) to
   GitHub for the live site to pick all of this up.

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

- **Meal cards collapse to a compact row by default** (per explicit ask —
  6 always-fully-expanded cards per day, each with a staple row, dish list,
  status buttons, and a whole thread, was too heavy). Every card starts
  collapsed to just its tab header plus either a thin "+ Add dish" prompt
  (`slotHasData()` false — no dish, default staple/status, no messages) or
  a compact one-line summary (dish name(s), a colored status pill) once
  it's actually been used. **The one exception: a card with an unread
  message auto-expands on load**, so a message from the helper or another
  family member is never missed without opening anything. Tapping the tab
  (which doubles as a toggle button — cursor:pointer, a `▸`/`▾` chevron) or
  the collapsed summary row itself expands/collapses a card into the full
  view. Expand state is tracked in `cardExpandState`, keyed
  `` `${dateIso}::${slotKey}` `` (same UI-only, never-synced-to-Firestore
  pattern as `videoPlayingState`) — once a card is explicitly toggled open
  or closed this session, that choice sticks even if you switch days and
  come back to it; before any explicit toggle, `isCardExpanded()` falls
  back to "has an unread message" as the default. Don't move this state
  into Firestore — it's meant to be per-viewer, not shared (two people
  looking at the same day shouldn't have their card open/closed state
  forced to match).
- Warm parchment aesthetic: dotted background, Fraunces serif headers, Karla
  sans body, dashed dividers — same visual language as `wg-groupbuy`'s
  receipt-style dashboard, adapted to a softer household-app palette (see
  `:root` CSS vars in either page for the full light/dark token set).
- **Light/Dark/Auto is a segmented pill control** (`.theme-switch`/`.theme-seg`),
  not a single cycling icon button — this was an explicit revision after an
  icon-only version was tried first. **Later reverted back to icon labels**
  (☀️/🌙/🖥️) instead of the text "Light"/"Dark"/"Auto" — Tagalog's longer
  words (Maliwanag/Madilim/Awtomatiko) were pushing the segmented control
  wide enough to squeeze the EN/中文/TL language buttons, wrapping "中文"
  onto two lines. The pill *shape* stayed (still three tappable segments,
  still highlights the active one), only the label content changed from
  text to emoji; the localized name is still set as each button's `title`
  attribute (via `data-i18n-title`, a `syncLangUI()` sweep alongside
  `data-i18n`/`data-i18n-ph`) for anyone hovering on desktop. The Auto icon
  was originally a half-moon/sun (🌗) and was swapped to a computer (🖥️) per
  explicit ask, since "system default" reads more clearly as a device icon
  than as a lunar phase. If this control is touched again, don't
  reintroduce text labels for the segments — that's the whole reason for
  this change.
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
  visibly flagged rather than silently blank. **Originally this only lasted
  the browser session unless also hand-edited into `recipes.json`** — fixed
  by wiring `writeDish()` up to `recipeLibrary/library` in Firestore (see
  Firestore schema above), reported after a video-link fix made on one
  device wasn't showing up on another. Every add/edit now syncs live to
  every device, same as the rest of the app.
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

- **Sent-message "translation" no longer repeats the original text.** There's
  still no real translation service wired in, so a message you send gets a
  placeholder `text_tl` — but it used to be `${text} (halimbawang salin)`,
  which just echoed your own message back, redundant with the original
  shown right above it. It's now a fixed status line, "(Awtomatikong salin
  — wala pang aktibong serbisyo)" — **hardcoded in Tagalog, not run through
  `t()`**, because `text_tl` is supposed to always be the Tagalog side for
  the helper regardless of the current UI language toggle (this is the same
  toggle-leaking-into-message-content bug flagged earlier in this file —
  don't reintroduce it here either).
- **Translation runs on MyMemory (free, no key), not paid Google Cloud
  Translation** — switched after deciding not to set up billing just for
  this. MyMemory (https://mymemory.translated.net) is a public, free,
  no-signup translation API — plain `fetch()` GET request, no key, no auth.
  The trade-off: it doesn't detect language for you like Google's paid API
  did, so language is now guessed locally instead (`detectLangHeuristic()`)
  — no network call, always returns `'en'`/`'zh'`/`'tl'`:
  - Chinese is detected by the presence of any CJK character — unambiguous,
    since neither English nor Tagalog uses that script.
  - Between English and Tagalog (both Latin-script), the message counts as
    Tagalog if it contains at least one word from `tagalogMarkerWords` or one
    of the short `tagalogMarkerPhrases` as a substring of the whole message.
    Both are loaded from `tagalog-markers.json` at boot (206 words — the
    stopwords-iso project's official 147-word Tagalog stopword list plus our
    own curated colloquial/texting words like `po`, `daw`, `salamat`,
    `kumusta` that a formal stopword list wouldn't include — plus 19 short
    phrases like `"walang anuman"` or `"kumusta ka"` that only make sense
    checked as a unit, not split into individual words). If the fetch fails,
    a small hardcoded fallback set baked into `meal-dashboard.html` is used
    instead — detection still works, just with fewer words to match against.
    A word is also checked with a trailing `"ng"` stripped off before
    matching (`kaming`→`kami`, `tayong`→`tayo`, `silang`→`sila`), since
    Tagalog's `-ng` linking particle attaches straight onto pronouns and
    this covers that whole family without enumerating every linked form by
    hand — safe against false positives, since a stripped English word like
    `"morning"` → `"morni"` isn't a marker either way. Otherwise the message
    is assumed English. **A real miss this caught:** "Mayroon kaming isda at
    bok choy." went completely untranslated (no marker word matched even
    though it's Tagalog) until `mayroon` was added and the `-ng` stripping
    was added for `kaming` — this is also what prompted pulling in the full
    stopwords-iso list rather than continuing to hand-maintain a short one.
    If another real message goes untranslated, the fix is almost always
    adding the missing word/phrase to `tagalog-markers.json` (no code
    change needed), not a bug in the translation call itself.
  - This is a heuristic, not real language detection — short or unusual
    messages can still be misclassified (a Tagalog sentence using none of
    the listed words/phrases, in none of their handled inflected forms,
    would read as English). Tested against the
    actual messages from early testing ("What's for lunch tomorrow?" → en,
    "Marunong akong magluto ng omelet." → tl, "我不知道。有时间吗？" → zh,
    "Kumusta ka?" → tl) and all classified correctly, but it's not
    bulletproof — if misclassification becomes a real problem, the fix is
    adding to `tagalog-markers.json` or switching to a paid detection API.
  - Target is still fixed at English for everyone, same reasoning as
    before: you type English only, your wife types English or Chinese, the
    helper types English or Tagalog — everyone already understands
    English, so there's no need to translate into three languages or vary
    the target by who's viewing.
  A sent message stores `{ text, origLang, translatedEn }` — unchanged
  shape from before, just `origLang` now comes from the local heuristic
  instead of an API call, and `translatedEn` comes from MyMemory instead of
  Google:
  - `text` — exactly what was typed.
  - `origLang` — always `'en'`, `'zh'`, or `'tl'` now (the heuristic always
    returns a guess; it's not "unknown" the way a failed API call used to
    be, so a `null` value should no longer appear in fresh messages).
  - `translatedEn` — fetched via one `translateText()` call to MyMemory,
    only when `origLang !== 'en'`. `null` if the MyMemory call fails or
    returns nothing usable — rendering falls back to "Translation
    unavailable" in that case, same as before.
  - **Rendering is still the same for every viewer**, not per-viewer — the
    translated box shows `translatedEn` whenever `origLang !== 'en'`,
    regardless of the viewer's own EN/中文/TL toggle. (This was already the
    design going into this change; only the translation backend changed.)
  - **Rate limit**: MyMemory's anonymous tier is roughly 5,000 words/day
    per IP address — trivial for household chat volume, but if it's ever
    hit, appending `&de=<an email address>` to the request URL in
    `translateText()` raises the anonymous limit (MyMemory's documented
    mechanism — no account needed, just an email string in the query).
  - No setup needed — no API key, no billing, no Google Cloud Console.
    `translateText()`/`detectLangHeuristic()` work out of the box.
  - Old messages (the original `text_zh`/`text_tl` shape, from before any
    of this and the original hand-typed helper/you demo pairs) still
    render exactly as before — the renderer branches on whether `m.text`
    exists.
- **Chat bubble alignment is per-viewer too, not just the label.** Fixing
  the "You" label (above) didn't fix alignment — a message from another
  family member's device still rendered right-aligned like your own,
  because alignment was keyed off `m.from === 'you'` while the label was
  already keyed off `senderId`. Both now go through one shared
  `isOwnMessage(m)` helper: a message gets the `.mine` CSS class (and
  right-alignment) only on the device that actually sent it; everyone
  else's device shows that same message left-aligned with the sender's
  name, same as a helper message. `m.from` itself is untouched and still
  drives the Chinese/Tagalog pairing and translation-box color — those
  don't depend on who's viewing.

- **"Reset this week" button (🗑️, next to the bell) — testing only, remove
  before "live"/deployed.** Originally only wiped messages; now resets the
  *current week only* back to a blank slate entirely — every date's
  dishes/staple/status (`mealPlans/{weekId}` → `{}`) **and** every message
  (`mealThreads/{weekId}` → `{}`) — leaving other weeks and the shared
  recipe library (`recipeLibrary/library`) untouched. Confirms via a native
  `window.confirm()` before doing anything, because **it writes straight to
  Firestore**, so it resets the shared/live copy for every device currently
  looking at this week, not just the one that clicked it — this isn't a
  "reset my local view" button. This is explicitly a testing convenience,
  not a feature for the finished household tool — remove the button (and
  its handler/i18n strings) entirely before handing this off as "live,"
  since anyone with the page open can wipe an entire week's plan and
  messages for everyone with two taps.

- **Unread bell/badge only ever counted `from:"helper"` messages —
  meaningless once real people all send as `"you"`.** Two bugs compounded:
  the send handler never set `unread: true` on a new message at all, and
  every unread-counting site (`unreadForMeal()`, the notification popover,
  `jumpToMealThread()`'s mark-as-read, and the Home Hub's
  `subscribeMealsBadge()`) filtered on the hardcoded `from === 'helper'`
  role. Since the multi-user redesign means your wife and the helper also
  send as `from:"you"` (just with their own `senderName`), none of that
  ever matched — the bell and Home Hub badge would never have lit up for
  anything anyone actually typed. Fixed by: (1) setting `unread: true` on
  every newly sent message, and (2) switching every counting/marking site
  to `!isOwnMessage(m)` instead of `from === 'helper'` — "unread" now means
  "not sent by this device," which is what actually matters once anyone
  can be on either side of a conversation. `index.html`'s badge duplicates
  an `isMine()` check matching `isOwnMessage()` exactly (the two pages
  don't share a script file) — **keep them in sync if either changes**.

- **Send button was getting squeezed off-screen** on narrower viewports —
  classic flexbox bug: `.msg-input input` had `flex:1` but no `min-width:0`,
  so a flex child's default `min-width:auto` stopped it from shrinking
  below its own content width, pushing the Send button past the edge of
  the card instead of the input just getting narrower. First fix was
  `min-width:0` plus a fixed 38×38px icon button ("➤"); **the button was
  later removed entirely per explicit ask** (still overlapping the
  on-screen keyboard on mobile even at a fixed size) — now there's no Send
  button at all, just the input and a small italic "press Enter to send"
  hint underneath (`.send-hint`, localized as `sendHint`). The send logic
  lives in `sendMessageForSlot(slotKey)`, called only from the `Enter`
  keydown listener on the message input (Shift+Enter is left alone, though
  a single-line `<input>` has no newline to insert regardless); the input
  is disabled during the translation `await` to prevent a double-send, then
  a fresh (enabled, focused) input is created when `renderMeals()` rebuilds
  the card right after.

- **Notification popover could render partly off-screen to the left.** Its
  CSS positioned it with `right:0` relative to the bell button itself — a
  260px-wide popover anchored that way assumes the bell sits near the right
  edge of the header, which stopped being reliably true once the header
  also grew theme icons and the clear-messages button. On a packed/narrow
  screen, `bellRect.right - popoverWidth` could land well left of the
  viewport's own left edge, with no way to scroll to the cut-off part.
  Fixed in `openNotifPopover()`: it now measures the bell and popover with
  `getBoundingClientRect()` every time it opens and clamps the computed
  position to stay within `8px` of either viewport edge, converting back to
  a position relative to the bell (its `offsetParent`) before applying it.
  The CSS also gained `max-width: calc(100vw - 16px)` as a backstop so the
  popover itself can never be wider than the viewport regardless.

## Troubleshooting / lessons already learned

- **"Unread" was a single shared boolean on the message — one device
  marking it read marked it read for every device.** `unread: true` was
  set once at send time and flipped to `false` in Firestore by whoever
  first read it (tapping a notification, or — once card-collapse landed —
  simply expanding the card). Since that field lives in the shared
  `mealThreads` doc, the flip synced to every viewer immediately: someone
  could open a message on their phone, and it would silently disappear
  from the bell/badge on everyone else's device too, even though they'd
  never actually seen it — the exact bug report that prompted this fix.
  Replaced with `readBy`, an array of viewer ids (`getUserId()`) that a
  message accumulates as each person actually reads it —
  `isUnreadForMe(m)` (in both `meal-dashboard.html` and, as `isMine`'s
  sibling, `index.html`'s badge subscriber) only returns true if *my*
  id isn't in that list yet, so my read state and your read state are
  now genuinely independent even though they live in the same synced
  document. Marking-as-read now happens in three places, all via
  `markThreadReadForMe()`: tapping a notification popover item, manually
  expanding a collapsed card, and sending a reply into a thread (replying
  implies you've seen what's already there). Legacy messages from before
  this fix have no `readBy` field, so they read as unread-for-everyone
  until each person's device actually opens them — a one-time bump the
  first time this code runs against old data, not a bug. **Don't go back
  to a single shared `unread` flag** — any "mark as read" action must only
  ever add to a message's own `readBy` list, never overwrite the read
  state for a viewer other than the one performing the action.

- **The day-strip used to be purely cosmetic.** `mealData`/`threadData` were
  flat, keyed only by meal slot for the whole week, so clicking Mon/Tue/
  Wed/... re-rendered the exact same dishes/status/thread every time —
  nothing was actually keyed by date. Fixed by nesting Firestore's
  `mealPlans`/`mealThreads` docs (and their local mirrors) under
  `[dateIso][slotKey]` (see Firestore schema above): an untouched date has
  no entry and renders empty via `peekDayPlan()`/`peekDayThreads()`; only
  `writeMealSlot()`/`writeThread()` (called from an actual add/edit/send)
  persist a date, which also drives the small dot shown on a day-tab that
  has data (`dayHasData()`). Don't flatten this back to slot-only keys —
  and note this means any messages/plan data written under the old flat
  shape won't show up anymore (see Setup above).

- **Home Hub showed only the ghost placeholder, no Meals card** — reported
  once after the Firebase/name-entry changes landed; root cause wasn't
  pinned down with certainty (no console access to the live failure), but
  the fix applied either way: the ghost/placeholder card was removed
  entirely per explicit request, and `hub-cards.json` fetch failing or
  returning nothing usable now falls back to a hardcoded Meals-only card
  (`FALLBACK_HUB_CARDS` in `index.html`) instead of silently leaving the
  grid empty. If a blank-hub report ever recurs, checking the browser
  console on the live page for a fetch/CORS/404 error is the next step.

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
- **A dedicated fullscreen button (`.video-fullscreen-btn`) sits on top of
  every playing video embed**, per explicit ask — the YouTube player's own
  fullscreen control is small and easy to miss at this card's embed size.
  It calls `requestFullscreen()` (with vendor-prefixed fallbacks) on the
  `<iframe>` element itself, not the player inside it — a cross-origin
  iframe's internal player can't be reached directly from the parent page,
  but fullscreening the iframe element achieves the same visual result.
  `fullscreen` is listed explicitly in the iframe's `allow` attribute
  alongside the legacy `allowfullscreen` boolean, since relying on
  `allowfullscreen` alone isn't consistently honored across browsers.
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
