# Cross-device progress sync for Italian Flashcards

## Problem

The Learn mode's "mastered card" progress currently saves to the browser's
`localStorage` (see `saveLearnProgress` / `loadLearnProgress` in
`italian_flashcards.html`). This only persists on one browser/device — it
does not follow the user across devices or browsers.

## Goal

Let the user sign in with their Google account and have Learn mode progress
follow them across any device/browser where they sign in, while still
working offline / signed-out via the existing localStorage fallback.

## Non-goals

- Syncing Quiz mode or Flashcard mode progress — those are in-session-only
  progress bars today (never persisted, even locally) and stay that way.
- Multi-user features (sharing progress, leaderboards, etc.).
- Removing or changing the existing `window.storage` (Claude-artifact)
  branch already present in the code — it's left as-is, just extended with
  a new option.

## Architecture

The app remains a single static HTML file with no build step and no custom
backend. Two Firebase products are added via the Firebase JS SDK, loaded
as an ES module from Google's CDN:

- **Firebase Authentication** — Google sign-in (popup flow), giving each
  user a stable `uid`.
- **Cloud Firestore** — one document per user at `progress/{uid}`, shape:
  ```json
  { "masteredIds": ["...", "..."] }
  ```

Firebase project setup (console steps, Spark/free plan) is done manually
by the user; the resulting config object (`apiKey`, `projectId`, etc.) is
pasted into the HTML's Firebase init block. No secrets beyond this public
client config are involved — Firebase web config is not sensitive, access
control is enforced by Firestore security rules (below).

## Components

- **Auth UI**: a "Sign in with Google" button near the existing Learn mode
  controls. When signed in, it's replaced with the user's avatar/email and
  a "Sign out" control.
- **Storage layer**: `saveLearnProgress()` / `loadLearnProgress()` /
  `resetLearnProgress()` gain a third branch alongside the existing
  `window.storage` / `localStorage` split:
  1. `hasArtifactStorage()` → `window.storage` (unchanged)
  2. else, signed in → Firestore doc `progress/{uid}`
  3. else → `localStorage` (unchanged)
- localStorage continues to be written as a local cache/fallback any time
  progress changes, regardless of sign-in state, so nothing is lost if a
  Firestore write fails.

## Data flow

- **On sign-in**: read `progress/{uid}` from Firestore.
  - If the doc exists, it becomes the source of truth — replaces the
    in-memory `learnMasteredSet` (and refreshes localStorage cache).
  - If it doesn't exist (first sign-in on this account) and there is local
    progress already, push the local set up as the initial cloud copy.
- **On mastered-card change**: update in-memory state immediately (as
  today), then:
  - write to `localStorage` synchronously (cache/fallback), and
  - if signed in, write to Firestore, debounced ~1s so rapid successive
    taps don't trigger a write per tap.
- **Sign out**: no data deletion. App reverts to localStorage-only
  behavior using whatever is currently in localStorage.

## Error handling

- Firestore write fails (offline, quota, transient error): caught and
  logged to console (matches existing `catch(err){ console.error(...) }`
  pattern in the code); localStorage write already succeeded so the user
  doesn't lose progress. No blocking UI error — sync is best-effort.
- Firestore read fails on sign-in: falls back to whatever is currently in
  localStorage/in-memory rather than blocking Learn mode from loading.
- Sign-in popup blocked or cancelled by the user: no state change, app
  stays in signed-out/localStorage mode.

## Security

Firestore security rules restrict each document to its owner:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /progress/{uid} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
  }
}
```

## Testing

No automated test suite exists for this static HTML app; verification is
manual:

1. Sign in on device/browser A, mark several cards mastered, confirm they
   appear in the Firestore console under `progress/{uid}`.
2. Sign in with the same Google account on device/browser B, confirm the
   same mastered cards appear without any manual action.
3. Mark additional cards mastered on B, reload A, confirm A picks up the
   change.
4. Go offline (dev tools offline mode) while signed in, mark a card
   mastered, confirm it saves locally without a JS error; go back online
   and confirm it syncs up on the next change (or next load).
5. Sign out, confirm the app falls back to localStorage-only behavior
   exactly as it does today.
