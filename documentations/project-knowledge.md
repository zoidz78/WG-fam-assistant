# WG Family Assistant — Project Knowledge & Setup Guide

**Read this file first in any new chat on this project.** It's the single source
of truth for how this project works and what to do next. This file was rewritten
2026-09-17 to reflect the current state after several rounds of fixes — if an
older copy of this file is floating around (chat history, a stale download), this
version supersedes it.

## What this project is

A household hub for the family + their live-in helper. Static site, hosted on
**GitHub Pages** (a separate GitHub account from `wg-groupbuy`, uploaded manually
by the user — this session has no push access to it, so **every file this
session edits must be re-uploaded by the user before it's live**). No build
step — plain HTML/CSS/JS files plus JSON data files, with Firestore for
anything live/cross-device.

**Pages:**
- `index.html` — Home Hub landing page. Reads `hub-cards.json` for what cards
  to show.
- `meal-dashboard.html` — "What's Cooking": a weekly meal planner with dynamic
  meal cards, a recipe library, and a unified translated chat thread with the
  helper.
- `admin.html` — Admin page (2026-09-17). PIN-gated (same PIN, `1117`) at the
  page level — the whole page is blocked behind a full-screen PIN prompt
  until unlocked, not just individual actions. Currently has one working
  section: viewing/renaming/deleting `familyMembers` records, for cleaning up
  old test-device entries. Explicitly scoped to grow — more database-editing
  tools will be added here later (a "More admin tools" placeholder card
  already sits below the family-members one). Linked from a new "Admin" card
  on the Home Hub (`hub-cards.json`, id `admin`, icon 🛠️).
- `talk-bridge.html` — "**Talk Bridge**" / "**语桥**" (2026-09-17). A
  standalone, full-page translated chat — same MyMemory translation engine
  and Mandarin/English/Tagalog language-detection logic as the meal chat,
  but pulled out into its own page with much bigger fonts (~19px message
  text vs ~13.5px in the meal chat's compact card), built specifically so a
  less tech-savvy family member (this was requested for a mother-in-law) can
  read messages to/from the helper comfortably. Not tied to meal planning or
  a specific week — one ongoing conversation (`talkBridge/main`). Has its
  own PIN-gated "clear chat" (🗑️ in the header) and reuses the same name
  modal + cross-device identity-merge-with-PIN flow as the other pages. Card
  sits on the Home Hub between "What's Cooking" and "Admin" (`hub-cards.json`,
  id `talkbridge`, icon 🌉), with its own unread-message badge like the meal
  chat's.
  - **"Clear chat" archives, it doesn't truly delete** (2026-09-17, explicit
    request): before emptying `talkBridge/main`, the current messages are
    written to a new doc in `talkBridgeArchive/{archiveId}` (`{messages,
    clearedBy, clearedById, clearedAt}`), which is append-only in
    `firestore-rules.md` (same as `auditLog` — create allowed,
    update/delete blocked) so an archived conversation can't be edited or
    erased afterward either. The `auditLog` entry for the action
    (`clearTalkBridge`) only records `{archivedAs: archiveId,
    messageCount}`, not the full message text — the actual content lives in
    the archive doc, not the audit trail. If the archive write fails, the
    clear is aborted entirely (nothing is wiped without a successful backup
    first) and no audit entry is logged for that attempt. There's currently
    no in-app UI to browse `talkBridgeArchive` — retrieving an old
    conversation means looking it up in the Firebase Console.

## Files

| File | Purpose |
|---|---|
| `index.html` | Home Hub landing page. Fetches `hub-cards.json`, falls back to `FALLBACK_HUB_CARDS` if that fails. |
| `hub-cards.json` | List of Home Hub cards: `{id, href, icon, text: {en, zh}}`. |
| `meal-dashboard.html` | The meal planner + recipe library + chat. See sections below. |
| `talk-bridge.html` | Standalone big-font translated chat page ("Talk Bridge" / "语桥") — see the page list above. |
| `recipes.json` | Seed data for the Firestore recipe library (see Recipe library section) — **not the live source once Firestore has data**. |
| `firestore-rules.md` | Firestore security rules for all collections used by this app. |
| `README.md` | User-facing docs for a non-technical maintainer — **stale, still describes the old fixed-6-slot flow; needs a rewrite before handoff.** |
| `admin.html` | PIN-gated admin page — see the "Admin page" section above. |
| `project-knowledge.md` | This file. |

## Current visual design — WeChat-style, no theme toggle

- **White/near-white background** (`--bg:#F7F7F7`, `--paper:#FFFFFF`) with an
  **orange sticky top banner** (`--banner-bg:#F2711C`, white banner text/icons).
  The `--teal` CSS variable name was kept for continuity with older code even
  though its value is now orange, not teal — don't be confused by the name.
- **No light/dark/auto theme toggle.** Removed entirely (2026-09-17) — the
  ☀️/🌙/🌗 buttons, `applyTheme()`/`themePref`/`localStorage['wg-theme']`, and
  the `prefers-color-scheme`/`data-theme` CSS blocks are all gone. Both pages
  render one fixed light palette. Don't reintroduce this unless asked again.
- System font stack, no Google Fonts (unreliable behind China's firewall):
  `--font-body` (system sans) / `--font-head` (system serif, `Songti SC` etc.).
- **User's name shows as a bold pill badge** in the header, left of the
  language pills (`.greeting`/`#greetingBtn`), on BOTH pages — larger and more
  prominent than the original small text version. Tapping it reopens the name
  modal. Comes from `localStorage['wg-username']`, shared across both pages
  since they're the same origin.
- Language toggle is two pills, 中文/EN only (Tagalog dropped from UI chrome
  in the WeChat redesign — see below). Default is `zh`.

## What's Cooking page layout (top to bottom)

Sticky orange header → day strip (contextual/navigational only, doesn't gate
which cards render — the plan is week-level, not per-day) → **meal cards**
→ **chat card** (`.chat-card`, moved to sit AFTER the meal cards, right above
the sticky input bar, per explicit request) → sticky bottom chat input bar.

## Meal cards (dynamic, not fixed slots)

`mealPlans/{weekId}` stores `{ cards: [{id, audience:'adult'|'kids', mealType,
staple?, dishIds:[{id}], status}] }` — a card only exists if someone added it
via the "+ Add a meal" ghost card (expands an inline draft: pick audience →
meal type → recipe). No fixed 6-slot grid.

## Chat (unified, translated, PIN-gated deletion)

- **One thread for the whole week**, not per-meal-slot: `mealThreads/{weekId}`
  = `{ messages: [...] }`.
- **Multi-language translation**: every message gets a `translations` object
  keyed by whichever of `en`/`zh`/`tl` it ISN'T written in, via MyMemory
  (free, no-key translation API).
  - **Language detection is ratio-based, not "any CJK char = Chinese."**
    `detectLangHeuristic()` only classifies a message as Chinese if Chinese
    characters make up >40% of it — a fix for code-mixed messages (e.g.
    mostly-English with one Chinese phrase mixed in) that used to get
    entirely misclassified as Chinese, which then skipped generating a
    Chinese translation for the English majority of the text.
  - **Chinese↔Tagalog pairs route through English** (`translateViaBestPath()`):
    MyMemory's direct zh↔tl pair was low quality (came back barely-translated,
    sometimes reading like English). Any pair involving English is still a
    single direct call, since that's the case MyMemory handles well.
  - **Known limitation, not yet solved**: a single message is still
    translated as ONE language end-to-end. A genuinely code-mixed sentence
    (half Chinese, half Tagalog, say) gets a big quality improvement from the
    pivot fix above, but isn't split per-segment and translated part-by-part
    — that would need real per-segment language detection, out of scope for
    the current free-API setup.
- Chat card header has a 🗑️ clear-all button — **PIN + double-confirm +
  audit-logged**, see below.

## PIN + double-confirm + audit log (destructive/edit actions)

Added 2026-09-17 as a friction check between household members (**not** a
real security boundary — the PIN is a plain string in client-side JS, visible
to anyone who views page source; fine for "don't fat-finger a delete between
family members," not fine against an outside actor).

- **PIN is `1117`.**
- **Delete a meal card, remove a single dish from a card, and clear the whole
  chat thread** all go through the same gate: PIN prompt → a second, separate
  `window.confirm()` "are you sure" dialog → only then does the action run.
  Wrong PIN shows an inline error and doesn't proceed.
- **Editing an existing recipe (✏️)** is different by explicit request: the
  pencil button opens the edit modal **immediately, no PIN** — the PIN is
  only asked when you hit **Save**, with **no extra confirm dialog** (the
  Save click itself is the confirmation). Adding a brand-new dish (not
  editing one) needs no PIN at all.
- **Every one of the above (except opening the edit modal, and adding a new
  dish) writes an entry to Firestore's `auditLog` collection**: `{action,
  details, weekId, by (display name), byId (device id), ts}`. Actions logged:
  `deleteCard`, `removeDish`, `clearMessages`, `editDish`. `auditLog` is
  **append-only** in `firestore-rules.md` (create allowed, update/delete
  blocked) so the trail can't be edited or erased after the fact.
- Implementation: `requestPinThenConfirm(action)` — `action` is
  `{confirmText?, onConfirm, auditAction?, auditDetails?}`. `confirmText`
  omitted skips the second confirm step (used for the edit-save case).
  `performDishSave()` holds the actual dish-mutation logic, called either
  directly (new dish) or after a successful PIN check (editing one).

## Cross-device identity merge (same name, PIN-gated)

Added 2026-09-17, on both `index.html` and `meal-dashboard.html` (the name
modal exists identically on both pages, sharing `localStorage['wg-username']`
and `localStorage['wg-userid']` since they're same-origin).

**Problem it solves**: if someone types the same name (e.g. "Effendy") on a
second device, that device would otherwise get its own `wg-userid` and create
a second, duplicate `familyMembers` record — splitting that person's chat
history/attribution across two "identities."

**Behavior**: when the name modal is saved, `findExistingMemberByName(name)`
queries `familyMembers` (case-insensitive, trimmed match, excluding the
current device's own id) for an existing record with that name.
- **No match** → saves normally, no PIN, exactly as before.
- **Match found** → gated by the same `requestPinThenConfirm()` PIN prompt
  used elsewhere (PIN `1117`), but with **no second `window.confirm()`**
  (`confirmText` omitted) — entering the correct PIN and hitting Continue is
  itself the confirmation, same pattern as the recipe-edit-save flow. On
  correct PIN, the **current device's `localStorage['wg-userid']` is
  overwritten to the existing member's id** (`existing.id`), so this device
  now shares that person's identity — same `familyMembers` doc, same
  attribution in chat (`isOwnMessage()` will match on both devices going
  forward), no duplicate record created.
- Logged to `auditLog` as `claimIdentity` with
  `{name, claimedId, previousId}` (the device's old, now-abandoned
  `wg-userid`, in case anything needs to be traced back).

**Known limitation**: matching is by exact name string only — two different
people who happen to type the same name (e.g. two people both named "Mom")
would trigger this and could merge into one identity if either knows the PIN.
Acceptable per the user given this is a small household, but worth keeping in
mind if the family grows or names start colliding.

## Recipe library — Firestore-backed, real household dishes (not placeholders)

**This is the current, authoritative recipe set** — the original six
placeholder dishes (congee, Hainanese chicken rice, tomato egg, steamed fish,
braised pork, choy sum) were deleted entirely and replaced 2026-09-17 with
12 real household dishes, confirmed with the user before writing:

| id | English | Chinese |
|---|---|---|
| `udon_honey_wings` | Udon Noodles with Honey Fried Chicken Wings | 乌冬面配炸蜂蜜鸡翅 |
| `udon_jinjja_wings` | Udon Noodles with Jinjja Chicken Wings | 乌冬面配jinjja鸡翅 |
| `scrambled_eggs` | Scrambled Eggs | 炒蛋 |
| `tomato_egg_soup_rice` | Tomato Egg Soup with White Rice | 番茄鸡蛋汤配白饭 |
| `fried_chicken_ribs_rice` | Fried Chicken Ribs with White Rice | 炸鸡肋配白饭 |
| `squid_ink_noodles` | Squid Ink Noodles ("Black Noodles") | 墨鱼汁面（黑面面） |
| `korean_instant_noodles` | Korean Instant Noodles ("Korean Noodles") | 韩国泡面（韩国面面） |
| `braised_chicken` | Braised Chicken (Red-Braised) | 红烧鸡 |
| `braised_duck` | Braised Duck (Red-Braised) | 红烧鸭 |
| `blanching_technique` | Blanching (Meat Prep) | 焯水 |
| `steamed_fish` | Steamed Fish | 蒸鱼 |
| `stir_fried_vegetables` | Stir-Fried Vegetables | 炒蔬菜 |

Notes on this list:
- `红烧鸡/鸭` was explicitly split into two separate entries (`braised_chicken`
  / `braised_duck`) rather than one dish covering either protein.
- `blanching_technique` (焯水) has real instructions (given by the user): cold
  water + raw meat, 2–3 ginger slices, one knotted scallion, a splash of
  cooking wine, bring to a boil, skim off the blood foam, discard the water.
  It's technically a prep technique rather than a standalone meal, but was
  added to the recipe library as a reference card anyway per explicit
  request, marked "(Steps to be refined later.)" It won't make much sense
  assigned to a meal card the way an actual dish would — it's there for
  reference.
- Every other dish's `note` field is the placeholder text **"Recipe steps to
  be shared via chat."** — per explicit decision, detailed instructions for
  those dishes will be communicated through the chat feature rather than
  written into the recipe library up front. Update `recipes.json` (and the
  live Firestore doc — see below) as real instructions come in.
- All 12 have `video: false, videoId: null` — no video links yet.

### IMPORTANT — reseeding the live Firestore recipe library

`recipes.json` is only read as a **seed** the first time the
`recipeLibrary/main` Firestore document is created (see
`initRecipeLibrary()` in `meal-dashboard.html`) — once that document exists
with data in it, edits to `recipes.json` alone do **not** propagate to it.

**Since the live site already had the old 6 placeholder dishes seeded into
Firestore, replacing `recipes.json` in this repo is not enough on its own to
remove them from the live app.** To actually apply this dish overhaul to the
live site:

1. Re-upload the new `recipes.json` to GitHub Pages (for future fresh
   installs / the Firebase-not-configured fallback).
2. In the Firebase Console → Firestore Database → data, delete the
   `recipeLibrary/main` document (or clear its `recipes` field to `[]`).
3. Reload the live site once — `initRecipeLibrary()` will see the library is
   empty and reseed it from the new `recipes.json`, giving everyone the new
   12-dish list. (Any meal cards already assigned an old dish `id` like
   `century_congee` will show the "⚠️ Recipe not found" placeholder after
   this, since those ids no longer exist — worth checking current meal cards
   before doing this reset, in case anything needs re-picking.)

There is currently no in-app "reset recipe library to seed" button — this is
a manual Firebase Console step. If dish list overhauls like this become
routine, a future session could build one (PIN-gated, presumably).

## Firestore collections (all in `firestore-rules.md`, all open read/write
except `auditLog` which is append-only)

- `mealPlans/{weekId}` — `{ cards: [...] }`, week-level (Monday's ISO date).
- `mealThreads/{weekId}` — `{ messages: [...] }`, unified per week.
- `familyMembers/{memberId}` — `{ name, updatedAt }`, one per device
  (`localStorage['wg-userid']`).
- `recipeLibrary/main` — `{ recipes: [...] }`, one shared doc, seeded from
  `recipes.json` (see above).
- `talkBridge/main` — `{ messages: [...] }`, one shared doc, powers
  `talk-bridge.html`'s ongoing translated chat (not week-scoped).
- `auditLog/{entryId}` — `{ action, details, weekId, by, byId, ts }`,
  append-only, logs destructive/edit actions (see PIN section above).

## Design conventions to keep

- Kids cards: dashed border + `--kids-accent` (plum/lavender), no staple/rice
  selector, tab always reads "🧒 {Meal} · Kids" vs. "{Meal} · Adults" — both
  audiences labeled explicitly.
- Chat: helper messages left-aligned, your own right-aligned (per-viewer via
  `isOwnMessage()`, not a fixed `m.from==='you'` check — a message you send
  shows "You" only on your own device, your actual name everywhere else).
  Only the translation gets a colored box, never the original message.
- i18n: per-page `i18n = {en:{...}, zh:{...}}` + `t(key)` falling back to the
  key itself if missing — Tagalog stays OUT of these UI dictionaries (dropped
  in the WeChat redesign) but stays IN the chat-level translation/detection
  logic (`TAGALOG_MARKERS`, `detectLangHeuristic()`), since the helper can
  still type Tagalog.
- Recipe titles are bilingual (`title:{en,zh}`), switching display with the
  site's language toggle; a plain string `title` (legacy data) is treated as
  the same name in both languages. Notes/instructions stay single-language
  (English) — detail beyond that goes through chat.
- Stale-cache handling: `no-cache` headers, `?v=Date.now()` cache-busting on
  JSON fetches, forced reload on `pageshow` with `event.persisted`.
- YouTube embeds need `http://`/`https://` (not `file://`) to work at all —
  test via `python3 -m http.server`, not double-clicking the file.

## Files delivered to the user, not just saved to the Project

Every file this session edits is sent to the user as a real downloadable
file (`SendUserFile`) in addition to being saved into the claude.ai Project —
`project_write` alone does not hand the user a file, which was a real gap
flagged directly by the user earlier in this project's history. Keep doing
both on every edit.

**Quirk to know about**: `admin.html`, being a brand-new bare filename the
first time it was written to the Project, landed at the internal Project
path `claude/admin.html` rather than `admin.html` (the tool namespaces new
bare-filename docs this way). This does NOT affect the actual downloaded
file's name (still `admin.html`, correct for the live site) — it only means
a future `project_read`/`project_write` on this file needs the path
`claude/admin.html`, not `admin.html`, to find it.

## Not done yet

- `README.md` rewrite for the current flow (dynamic cards, unified chat,
  Firestore recipe library, PIN/audit system) — still describes the old
  fixed-6-slot / per-meal-thread behavior.
- No live-browser testing has been performed by this session at any point —
  only `node --check` (JS syntax), HTML tag-balance checks, and JSON
  validation. All real-world testing has been done by the user, iteratively,
  via screenshots.
- The Firestore recipe-library reseed (see above) is a manual step the user
  still needs to do in the Firebase Console — not yet confirmed done.
