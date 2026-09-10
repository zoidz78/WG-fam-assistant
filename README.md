# WG Family Assistant

A small household hub, hosted as a static site (GitHub Pages). Right now it's
one page — **Home Hub** (`index.html`) — linking to the first tool, **What's
Cooking** (`meal-dashboard.html`), a weekly meal planner + a message thread
with the helper. Changes to the meal plan, dish picks, and messages sync
live across every device — everyone sees the same thing in real time.

**Live site:** `https://<github-username>.github.io/<repo>/`

## Files

| File | Purpose |
|---|---|
| `index.html` | The Home Hub landing page. Reads `hub-cards.json` to know what cards to show — never hardcodes a card's data. Asks for your name on first visit. |
| `meal-dashboard.html` | The meal planner + message thread. Reads `recipes.json` for the list of dishes the helper knows how to cook. Always shows the current week (Monday–Sunday), computed from today's date, with each day planned independently — picking a dish or chatting about Tuesday's lunch doesn't touch any other day. Has a 🗑️ button next to the bell for wiping this week's messages — see "Clearing messages" below before tapping it. |
| `hub-cards.json` | List of cards shown on the Home Hub. Add an entry here to add a new section (e.g. chores, groceries) — no HTML/JS changes needed. |
| `recipes.json` | The shared recipe library — every dish the helper knows how to cook, with its cooking note and an optional video. Add an entry here to teach a new recipe. |
| `firestore-rules.md` | Firestore security rules for the live, cross-device parts of the app (staples, dish picks, status, messages, saved names). **Live** — already applied in the Firebase project this app uses. |
| `project-knowledge.md` | Technical notes for whoever's editing this project's code (not needed for everyday use — see below). |

## Your name

The first time you open the Home Hub, it asks for your name and remembers
it on that device. This is so messages you send in the meal thread show up
as "You" to you, but with your actual name to everyone else — so if more
than one person is chatting with the helper, it's clear who said what. Your
own messages sit on the right side of the thread; everyone else's (the
helper's, or another family member's) sit on the left — that's per-device,
so what shows as "your" side depends on which phone/browser you're on, not
who's speaking. Tap your name (top of the Home Hub) any time to change it.

## Messages & translation

Type your message and press **Enter** to send — there's a small reminder
under the box, and no separate button to tap.

Everyone can type in whatever language they're comfortable with — English,
Chinese, or Tagalog — message by message, no need to pick a language
first. The app guesses what language a message was typed in and, if it
wasn't English, shows an English translation underneath it, since everyone
using this understands English. An English message doesn't get a
translation line since there's nothing to translate. This works the same
way for every reader, no matter which of the EN/中文/TL buttons they
currently have selected — that toggle only changes the app's own labels
and buttons, not the conversation itself. Translation runs on a free
public service, so there's nothing to set up or pay for.

The language guess is a simple heuristic, not perfect — very short or
unusual messages occasionally get misread as the wrong language. If a
message shows "Translation unavailable" instead of an actual translation,
the translation service didn't respond that time — the original message
still sends and saves normally either way.

## Editing a recipe

Every dish in the meal plan has a ✏️ button next to it — tap it to fix or
fill in its cooking note or video link (handy for dishes that were added
without full details yet). This also works from the "choose a dish" list
when picking a dish for a meal slot. Like adding a brand-new dish from
inside the app, an edit made this way only lasts for your current browser
session — to make it permanent for everyone, edit `recipes.json` directly
(see below) and re-upload.

## Clearing messages

The 🗑️ button next to the bell icon on the meal dashboard deletes every
message in every meal slot **for the current week**, after asking you to
confirm. It's meant for testing, not everyday use — a few things worth
knowing before tapping it:

- It clears the **shared, live** copy — if the helper or another family
  member has the page open at the time, their messages disappear for them
  too, not just on your device.
- It doesn't touch the meal plan itself (dishes, staples, status) or any
  other week's messages — only this week's chat.
- There's no undo.

## Adding a new recipe

Open `recipes.json` and add an entry:

```json
{ "id": "unique_snake_case_id", "title": "Dish Name", "note": "Cooking instructions", "video": true, "videoId": "YOUTUBE_VIDEO_ID" }
```

- `id` must be unique — this is what a week's plan references.
- Set `"video": false, "videoId": null` if there's no video.
- `videoId` is just the 11-character YouTube ID (the part after `v=` or `youtu.be/`) — not a full link.

You can also add a recipe from inside the app itself (the "+ Add a new dish"
button), but that only lasts for your current browser session — to make a
new recipe permanent for everyone, add it to this file and re-upload.

## Adding a new Home Hub card

Open `hub-cards.json` and add an entry with `id`, `href`, `icon`, and the
`text` for each language (`en`/`zh`/`tl`). No other files need to change.

## Deploying

This is a static site with no build step. After editing any file, upload the
changed file(s) to GitHub (Add file → Upload files, or drag-and-drop) —
GitHub Pages updates the live site within about a minute. The site is set
up to avoid showing you (or anyone) a stale cached copy after an update, so
you shouldn't need to do anything extra on the viewing side.

`index.html` and `meal-dashboard.html` both connect to the same Firebase
project for live syncing — if that project's setup ever needs to change
(e.g. moving to a different Firebase project), both files' Firebase config
needs to be updated together, or the two pages will stop seeing each
other's changes.
