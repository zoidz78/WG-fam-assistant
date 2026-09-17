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
  icon-only version was tried first. **Later reverted back to icon labels**
  (☀️/🌙/🌗) instead of the text "Light"/"Dark"/"Auto" — Tagalog's longer
  words (Maliwanag/Madilim/Awtomatiko) were pushing the segmented control
  wide enough to squeeze the EN/中文/TL language buttons, wrapping "中文"
  onto two lines. The pill *shape* stayed (still three tappable segments,
  still highlights the active one), only the label content changed from
  text to emoji; the localized name is still set as each button's `title`
  attribute (via `data-i18n-title`, a `syncLangUI()` sweep alongside
  `data-i18n`/`data-i18n-ph`) for anyone hovering on desktop. If this
  control is touched again, don't reintroduce text labels for the segments
  — that's the whole reason for this change.
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
    Tagalog if it contains at least one word from `TAGALOG_MARKERS` (a
    ~40-word list of common short Tagalog function words: `ang`, `ng`,
    `hindi`, `wala`, `po`, `kumusta`, etc.) — otherwise it's assumed
    English.
  - This is a heuristic, not real language detection — short or unusual
    messages can be misclassified (a Tagalog sentence using none of the
    listed function words would read as English). Tested against the
    actual messages from early testing ("What's for lunch tomorrow?" → en,
    "Marunong akong magluto ng omelet." → tl, "我不知道。有时间吗？" → zh,
    "Kumusta ka?" → tl) and all classified correctly, but it's not
    bulletproof — if misclassification becomes a real problem, the fix is
    either a bigger marker word list or switching to a paid detection API.
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

- **"Clear all messages" button (🗑️, next to the bell) — testing only.**
  Wipes every message in every meal slot for the *current week only*
  (`mealThreads/{weekId}`) — leaves `mealPlans` (staples, dishes, status)
  and other weeks' message history untouched. Confirms via a native
  `window.confirm()` before doing anything, because **it writes straight to
  Firestore**, so it clears the shared/live copy for every device
  currently looking at this week, not just the one that clicked it — this
  isn't a "clear my local view" button. Safe to leave in for now since it's
  clearly labeled as testing-only, but worth removing (or hiding behind
  something less discoverable) before handing this off as a finished
  household tool, since anyone with the page open can wipe the week's
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
  the card instead of the input just getting narrower. Fixed by adding
  `min-width:0`, and separately made the button itself a fixed 38×38px
  icon button (a "➤" arrow, localized `send` string moved to `title`/
  `aria-label` instead of visible text) so it can never grow wide enough to
  cause this again regardless of content. The send logic itself was
  extracted into `sendMessageForSlot(slotKey)` so both the button's
  `onclick` and a new `Enter` keydown listener on the message input call
  the same code — pressing Enter now sends (Shift+Enter is left alone,
  though a single-line `<input>` has no newline to insert regardless).

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

## Major redesign — WeChat-optimized rebuild (2026-09-17)

Both `index.html` and `meal-dashboard.html` were rebuilt for WeChat's in-app
browser and a Mandarin-first household. **This is a breaking change** — read
this section before touching either file again.

### What changed and why
- **Tailwind CDN** (`https://cdn.tailwindcss.com`) added to both pages,
  alongside the existing CSS-custom-property color system (kept as-is —
  `--teal`/`--mustard`/`--brick`/etc. and the light/dark/auto toggle are
  unchanged). Static shell markup (header, tab switcher, modals) leans on
  Tailwind utility classes; the dynamically-rendered meal cards, dishes, and
  chat bubbles keep their original hand-written CSS classes (`.card`,
  `.dish`, `.msg`, etc.) — mixing the two was a deliberate choice to reuse
  already-correct rendering logic rather than rewrite it.
- **Google Fonts removed entirely** (`fonts.googleapis.com` is unreliable
  behind China's firewall, which defeats the point of a WeChat-optimized
  site). Fraunces/Karla are gone; both pages now declare `--font-body`
  (system sans, `-apple-system`/`PingFang SC`/`Microsoft YaHei` etc.) and
  `--font-head` (system serif, `Songti SC`/`STSong`/`SimSun`) so headers keep
  some visual distinction without any network font request.
- **Tagalog removed from the UI language toggle.** Both pages' `i18n` object
  now has only `en`/`zh`; the toggle is two pills (中文/EN), default language
  is now `zh` (`localStorage.getItem('wg-lang') || 'zh'`, was `'en'`).
  **This does NOT touch the chat's language detection/translation** —
  `detectLangHeuristic()` and `TAGALOG_MARKERS` are untouched, since the
  helper can still type Tagalog messages; only the interface chrome (button
  labels, day names, etc.) dropped Tagalog.
- **Sticky top banner** on both pages (`.app-header`, dark teal, `position:
  sticky; top:0`) replacing the old in-flow header — holds the page title,
  theme toggle, and language pills.
- **Meal cards are no longer 6 fixed always-shown slots.** The old
  `breakfast`/`breakfast_kids`/`lunch`/... fixed keys are gone. Instead
  `mealPlans/{weekId}` now stores `{ cards: [{id, audience, mealType, staple?,
  dishIds, status}] }` — a card only exists if someone added it. Tapping
  the "+ Add a meal" ghost card expands an inline **draft card** in place
  (not a modal) where you pick audience (Adult/Kids) then meal type
  (Breakfast/Lunch/Dinner), then search/pick a recipe from the same
  `recipes.json` library as before — selecting a recipe commits the card.
  Adding a *second* dish to an already-existing card still goes through the
  original pick-dish modal (unchanged flow), reusing `openPickDishModal()`.
  **The plan is still week-level, not per-day** (this matches the pre-existing
  design — the day strip was always cosmetic/contextual, never gating which
  cards render; this redesign didn't change that).
- **Chat is now ONE unified thread for the whole week**, not six separate
  per-slot mini-threads. `mealThreads/{weekId}` is now `{ messages: [...] }`
  instead of `{ [slotKey]: [...] }`. New messages carry a `ts` (send
  timestamp) field going forward, added specifically so a future feature
  could sort/merge more reliably — old messages have no `ts`. The chat
  lives behind a "💬 消息" tab (see below); a sticky bottom bar
  (`.chat-bar`, `position:fixed`) holds the text input + send button and is
  the only place with an always-visible input, shown only while that tab is
  active.
- **New tab switcher** ("🍲 餐单" / "💬 消息") replaces the old bell/
  notification-popover system entirely. The Chat tab shows a single unread
  count badge (no more per-meal breakdown, since messages aren't tied to a
  meal anymore); opening the tab marks everything read.
- **Migration on first load**: both `initFirestoreSync()` functions detect
  the old schema (`cards` missing on the plan doc / `messages` missing on
  the thread doc) and migrate automatically, once, in place:
  - `migrateOldMealPlan()` converts old slot data into cards — **only for
    slots that actually had a dish** (empty slots don't become empty ghost
    cards), consistent with the new "add only what's needed" philosophy.
  - `migrateOldThread()` flattens the six old per-slot message arrays into
    one array, concatenated in a fixed slot order (breakfast → breakfast_kids
    → lunch → lunch_kids → dinner → dinner_kids) since old messages have no
    timestamp to sort by properly — order across slots is best-effort only.
  - Old fields are left in the Firestore doc untouched (harmless, just
    unread by the new code) rather than deleted.

### Still true / unchanged from before
- `recipes.json` schema and semantics are untouched — a recipe is still
  audience/meal-type-agnostic; that association now lives on the *card*,
  same as it implicitly did on the old *slot*.
- YouTube embed behavior (video chip → tap → one-shot autoplay iframe,
  `autoplayDish` reset every render) is unchanged.
- Firestore rules, project config, and the three collections
  (`mealPlans`/`mealThreads`/`familyMembers`) are unchanged structurally —
  only the *shape of the documents* changed, not the collection names or
  security rules.
- The 🗑️ clear-messages button still exists (moved into the Chat tab's
  header area) and still wipes the *entire* current week's thread — now
  simply `writeThread([])` against the unified `messages` array. Still
  testing-only, still no undo, still worth hiding/removing before a final
  handoff — this was never addressed, just carried forward.
- `hub-cards.json` dropped its `tl` text per card (only `en`/`zh` remain);
  `index.html`'s card renderer already falls back to `en` if a language key
  is missing, so this is safe even if `currentLang` is ever stale.

### Not done in this pass
- `README.md` (the non-technical maintainer doc) was not rewritten for the
  new flow — it still describes the old fixed-6-slot / per-meal-thread
  behavior. Needs a pass before handing this off to a non-technical user.
- No live-browser testing was possible in this session (JS was syntax
  checked with `node --check` and HTML tag balance was checked
  programmatically, but nothing was rendered). Test via a local server
  (`python3 -m http.server`) before trusting this in production, same
  caution as ever about `file://` breaking both YouTube embeds and
  `fetch()`.
- The Tailwind CDN script (`cdn.tailwindcss.com`) is itself an external
  request — it loaded fine during development, but if WeChat's in-app
  browser in mainland China ever has trouble reaching it, the fallback is
  self-hosting Tailwind's compiled CSS instead of the Play CDN script (flagged
  here so a future session doesn't have to rediscover this).
