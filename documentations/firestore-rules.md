# Firestore Rules

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Stores the week's meal plan as a dynamic list of "cards" (each one
    // {id, audience: 'adult'|'kids', mealType, staple?, dishIds, status}) —
    // only meals the family actually added exist, there's no fixed set of
    // slots. Doc ID = that week's Monday date (e.g. "2026-09-08"), computed
    // by the dashboard from the real current date. Open read/write so the
    // dashboard can sync meal plan changes in real time for all viewers.
    match /mealPlans/{weekId} {
      allow read, write: if true;
    }

    // Stores one unified message thread for the whole week (field
    // `messages`, an array) — not split per meal slot. Doc ID = that
    // week's Monday date, same as mealPlans.
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

    // Stores the shared recipe library (field `recipes`, an array of
    // {id, title:{en,zh}, note, video, videoId}) in a single doc, `main`.
    // Not week-scoped — one doc shared by everyone. Seeded once from
    // recipes.json the first time this doc is created; every add/edit from
    // the UI (a new dish, a fixed video link) writes the whole array back
    // here, so it's live across every device instead of only lasting the
    // current browser session. Open read/write, same trust model as above.
    match /recipeLibrary/{docId} {
      allow read, write: if true;
    }
  }
}
```
