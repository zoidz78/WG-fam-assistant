# Firestore Rules

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Stores the week's meal plan, nested by date (one of that week's 7
    // ISO dates, e.g. "2026-09-10") then by meal slot (breakfast/lunch/
    // dinner x adults/kids): staples, dish-recipe references (by id,
    // resolved against recipes.json), and status (planned/cooking/missing/
    // done). A date only appears once something's actually been saved
    // against it — untouched days have no key.
    // Doc ID = that week's Monday date (e.g. "2026-09-08"), computed by the
    // dashboard from the real current date. Open read/write so the
    // dashboard can sync meal plan changes in real time for all viewers.
    match /mealPlans/{weekId} {
      allow read, write: if true;
    }

    // Stores the message thread (you <-> helper), nested by date then meal
    // slot within a week. Doc ID = that week's Monday date, same as
    // mealPlans. The "Clear all messages" testing button wipes this whole
    // doc back to {} (every date at once), not just the current day.
    match /mealThreads/{weekId} {
      allow read, write: if true;
    }

    // Stores each family member's chosen display name, one doc per device
    // (doc ID = a random ID generated client-side on first visit and kept
    // in localStorage as 'wg-userid'). Used so chat messages can show a
    // real name instead of a generic "You" label. Open read/write, same
    // trust model as the other two collections.
    match /familyMembers/{memberId} {
      allow read, write: if true;
    }

    // Stores the shared recipe library — a single fixed doc ("library"),
    // one top-level field per dish keyed by recipe id: { title, note,
    // video, videoId }. Seeded once from recipes.json the first time this
    // doc doesn't exist; after that, editing or adding a dish from the app
    // (the ✏️ button, or "+ Add a new dish") writes straight here and
    // syncs live to every device — recipes.json is no longer the ongoing
    // source of truth once this doc exists. Open read/write, same trust
    // model as the other collections.
    match /recipeLibrary/{docId} {
      allow read, write: if true;
    }
  }
}
```
