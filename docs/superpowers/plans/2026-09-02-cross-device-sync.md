# Cross-Device Progress Sync Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let Learn mode's mastered-card progress follow the user across devices by signing in with Google, backed by Firestore, while keeping the existing localStorage/artifact-storage behavior as offline fallback.

**Architecture:** `italian_flashcards.html` stays a single static file with no build step. It adds the Firebase **compat** SDK via `<script>` tags (not ES modules, to match the file's existing classic-script style), which exposes a global `firebase` namespace. Auth state is tracked in a `currentUser` variable updated by `firebase.auth().onAuthStateChanged`. The existing `saveLearnProgress`/`loadLearnProgress`/`resetLearnProgress` functions gain a Firestore branch, keyed by `progress/{uid}` documents, inserted between the existing `window.storage` (artifact) branch and the `localStorage` branch.

**Tech Stack:** Firebase Authentication (Google provider), Cloud Firestore — both via `firebase-app-compat.js`, `firebase-auth-compat.js`, `firebase-firestore-compat.js` from the Google CDN, pinned to v10.13.2.

## Global Constraints

- No build step — `italian_flashcards.html` remains directly openable/servable as a static file.
- Firebase compat SDK pinned to version `10.13.2` (exact CDN URLs given in Task 2).
- Firestore security rules must restrict `progress/{uid}` to `request.auth.uid == uid` (spec: Security section).
- Firestore writes triggered by mastered-card changes must be debounced ~1000ms (spec: Data flow).
- `localStorage` must continue to be written on every progress change as a cache/fallback whenever not using artifact storage, regardless of sign-in state (spec: Storage layer).
- No automated test suite exists for this project (static HTML, no test runner) — every task's verification is manual, in a real browser, per the spec's Testing section.

---

### Task 1: Firebase project setup (manual, console)

**Files:** none — this task produces values (`firebaseConfig`, confirms rules deployed) consumed by Task 2.

**Interfaces:**
- Produces: a `firebaseConfig` JS object (`apiKey`, `authDomain`, `projectId`, `storageBucket`, `messagingSenderId`, `appId`) that Task 2 pastes verbatim into the HTML.

- [ ] **Step 1: Create the Firebase project**

Go to https://console.firebase.google.com, click "Add project", name it (e.g. `italian-flashcards`), disable Google Analytics (not needed), click "Create project".

- [ ] **Step 2: Register a web app**

In the project overview, click the web icon (`</>`) to add a web app. Nickname it `italian-flashcards-web`. Do **not** check "set up Firebase Hosting" (the app is already hosted on GitHub Pages). Click "Register app".

- [ ] **Step 3: Copy the config object**

Firebase shows a `firebaseConfig` object like:
```js
const firebaseConfig = {
  apiKey: "...",
  authDomain: "italian-flashcards-xxxx.firebaseapp.com",
  projectId: "italian-flashcards-xxxx",
  storageBucket: "italian-flashcards-xxxx.appspot.com",
  messagingSenderId: "...",
  appId: "..."
};
```
Save these six values — Task 2 needs them.

- [ ] **Step 4: Enable Google sign-in**

In the console left nav: Build → Authentication → "Get started" → Sign-in method tab → click "Google" → toggle Enable → pick a support email → Save.

- [ ] **Step 5: Add authorized domain**

Still in Authentication → Settings → Authorized domains: confirm `londonzone.github.io` is listed (Firebase usually adds `*.web.app`/`*.firebaseapp.com` by default but not GitHub Pages domains). If it's not there, click "Add domain" and enter `londonzone.github.io`.

- [ ] **Step 6: Create Firestore database**

Build → Firestore Database → "Create database" → choose "Start in production mode" → pick any region (e.g. `us-central`) → Enable.

- [ ] **Step 7: Set Firestore security rules**

In Firestore → Rules tab, replace the contents with:
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
Click "Publish".

- [ ] **Step 8: Hand off config values**

Provide the six `firebaseConfig` values from Step 3 for use in Task 2.

---

### Task 2: Add Firebase SDK and initialize services

**Files:**
- Modify: `italian_flashcards.html:489-491` (insert SDK `<script>` tags and init block before the existing `<script>` at line 491)

**Interfaces:**
- Consumes: `firebaseConfig` values from Task 1.
- Produces: global `auth` (a `firebase.auth.Auth` instance) and `db` (a `firebase.firestore.Firestore` instance), available to all later `<script>` code in the file.

- [ ] **Step 1: Add the SDK script tags and init block**

In `italian_flashcards.html`, immediately before the existing `<script>` tag on line 491 (right after the closing `</div>` on line 489), insert:

```html
<script src="https://www.gstatic.com/firebasejs/10.13.2/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.13.2/firebase-auth-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.13.2/firebase-firestore-compat.js"></script>
<script>
const firebaseConfig = {
  apiKey: "PASTE_FROM_TASK_1",
  authDomain: "PASTE_FROM_TASK_1",
  projectId: "PASTE_FROM_TASK_1",
  storageBucket: "PASTE_FROM_TASK_1",
  messagingSenderId: "PASTE_FROM_TASK_1",
  appId: "PASTE_FROM_TASK_1"
};
firebase.initializeApp(firebaseConfig);
const auth = firebase.auth();
const db = firebase.firestore();
</script>
```

Replace each `"PASTE_FROM_TASK_1"` with the matching value captured in Task 1, Step 3.

- [ ] **Step 2: Verify the SDK loads with no console errors**

Serve the file locally (`python3 -m http.server 8000` from the project directory) and open `http://localhost:8000/italian_flashcards.html` in a browser. Open DevTools console.

Expected: no red errors mentioning `firebase` is undefined or `firebaseConfig`; the app's flashcards still render normally (this task only adds SDK plumbing, no visible UI change yet).

- [ ] **Step 3: Confirm `auth` and `db` are initialized**

In the DevTools console, type `auth` and `db` and press enter for each.

Expected: `auth` logs a `Auth` object (not `undefined`, no thrown error); `db` logs a `Firestore` object.

- [ ] **Step 4: Commit**

```bash
git add italian_flashcards.html
git commit -m "Add Firebase SDK and initialize auth/firestore"
```

---

### Task 3: Sign-in/sign-out UI and auth state listener

**Files:**
- Modify: `italian_flashcards.html:483-484` (insert sync status/buttons markup above the Learn mode progress bar)
- Modify: `italian_flashcards.html` end of main `<script>` block (add listener + button wiring, after the `document.getElementById('modeLearnBtn')...` block currently ending at line 1273)

**Interfaces:**
- Consumes: `auth` global from Task 2.
- Produces: a module-scoped `currentUser` variable (either a Firebase `User` object or `null`), read by Task 4's storage functions via a new `hasFirebaseUser()` helper.

- [ ] **Step 1: Add the sign-in/out UI markup**

In `italian_flashcards.html`, immediately before line 483 (`<div class="learn-stats">`), insert:

```html
    <div class="learn-sync">
      <span id="syncStatus" class="sync-status">Not signed in — progress saved on this device only.</span>
      <button class="ctrl-btn" id="signInBtn">Sign in with Google</button>
      <button class="ctrl-btn" id="signOutBtn" style="display:none;">Sign out</button>
    </div>
```

- [ ] **Step 2: Add `currentUser` state and the auth listener**

In the main `<script>` block, immediately before the `function cardId(idx){` line (currently line 1199), insert:

```js
let currentUser = null;

function hasFirebaseUser(){
  return !!currentUser;
}

auth.onAuthStateChanged(async (user) => {
  currentUser = user;
  const statusEl = document.getElementById('syncStatus');
  const signInBtn = document.getElementById('signInBtn');
  const signOutBtn = document.getElementById('signOutBtn');
  if(user){
    statusEl.textContent = `Signed in as ${user.email} — progress synced across devices.`;
    signInBtn.style.display = 'none';
    signOutBtn.style.display = 'inline-block';
  } else {
    statusEl.textContent = 'Not signed in — progress saved on this device only.';
    signInBtn.style.display = 'inline-block';
    signOutBtn.style.display = 'none';
  }
  if(learnCurrentIdx !== null){
    await loadLearnProgress();
    initLearnRound();
  }
});

document.getElementById('signInBtn').addEventListener('click', () => {
  auth.signInWithPopup(new firebase.auth.GoogleAuthProvider())
    .catch(err => console.error('Sign-in failed:', err));
});
document.getElementById('signOutBtn').addEventListener('click', () => {
  auth.signOut().catch(err => console.error('Sign-out failed:', err));
});
```

- [ ] **Step 3: Verify sign-in updates the UI**

Serve the file locally and open it (same as Task 2 Step 2). Switch to Learn mode. Click "Sign in with Google", complete the popup with a real Google account.

Expected: the popup closes, `#syncStatus` text changes to `Signed in as <your email> — progress synced across devices.`, the "Sign in" button hides, "Sign out" button appears. No console errors.

- [ ] **Step 4: Verify sign-out reverts the UI**

Click "Sign out".

Expected: `#syncStatus` reverts to `Not signed in — progress saved on this device only.`, "Sign in" button reappears, "Sign out" hides.

- [ ] **Step 5: Commit**

```bash
git add italian_flashcards.html
git commit -m "Add Google sign-in UI and auth state listener"
```

---

### Task 4: Firestore-backed save/load/reset

**Files:**
- Modify: `italian_flashcards.html:1208-1253` (the `saveLearnProgress`, `loadLearnProgress`, `resetLearnProgress` functions)

**Interfaces:**
- Consumes: `hasFirebaseUser()` and `currentUser` from Task 3; `db` from Task 2; existing `cardId(idx)`, `DECK`, `learnMasteredSet`.
- Produces: Firestore document `progress/{uid}` with shape `{ masteredIds: string[] }`, kept in sync with `learnMasteredSet`.

- [ ] **Step 1: Replace `saveLearnProgress`, `loadLearnProgress`, `resetLearnProgress`**

Replace the existing block (`italian_flashcards.html:1208-1253`, from `async function saveLearnProgress(){` through the closing `}` of `resetLearnProgress`) with:

```js
let firestoreSaveTimer = null;
function scheduleFirestoreSave(masteredIds){
  clearTimeout(firestoreSaveTimer);
  firestoreSaveTimer = setTimeout(async () => {
    try{
      await db.collection('progress').doc(currentUser.uid).set({ masteredIds });
    }catch(err){
      console.error('Could not sync progress to cloud:', err);
    }
  }, 1000);
}

async function saveLearnProgress(){
  const masteredIds = Array.from(learnMasteredSet).map(cardId);
  try{
    if(hasArtifactStorage()){
      await window.storage.set('learn-progress', JSON.stringify({ masteredIds }), false);
    } else {
      localStorage.setItem('italian-learn-progress', JSON.stringify({ masteredIds }));
      if(hasFirebaseUser()){
        scheduleFirestoreSave(masteredIds);
      }
    }
  }catch(err){
    console.error('Could not save progress:', err);
  }
}

async function loadLearnProgress(){
  try{
    let data = null;
    if(hasArtifactStorage()){
      const result = await window.storage.get('learn-progress', false);
      if(result && result.value) data = JSON.parse(result.value);
    } else if(hasFirebaseUser()){
      const doc = await db.collection('progress').doc(currentUser.uid).get();
      if(doc.exists) data = doc.data();
      if(!data){
        const raw = localStorage.getItem('italian-learn-progress');
        if(raw) data = JSON.parse(raw);
        if(data){
          await db.collection('progress').doc(currentUser.uid).set(data);
        }
      }
    } else {
      const raw = localStorage.getItem('italian-learn-progress');
      if(raw) data = JSON.parse(raw);
    }
    if(data){
      const idSet = new Set(data.masteredIds || []);
      learnMasteredSet = new Set(
        DECK.map((c,i)=>i).filter(i => idSet.has(cardId(i)))
      );
    }
  }catch(err){
    learnMasteredSet = new Set();
  }
}

async function resetLearnProgress(){
  learnMasteredSet = new Set();
  try{
    if(hasArtifactStorage()){
      await window.storage.delete('learn-progress', false);
    } else {
      localStorage.removeItem('italian-learn-progress');
      if(hasFirebaseUser()){
        await db.collection('progress').doc(currentUser.uid).delete();
      }
    }
  }catch(err){
    // ignore
  }
}
```

- [ ] **Step 2: Verify progress syncs up to Firestore**

Serve the file locally, open it, sign in, switch to Learn mode, answer a few cards correctly until at least one shows as mastered (mastered count > 0). Wait ~2 seconds for the debounce, then check the Firebase console: Firestore Database → data → `progress` collection → your `uid` document.

Expected: the document exists and its `masteredIds` array contains the card(s) you mastered.

- [ ] **Step 3: Verify progress loads on a second browser/profile**

Open the same locally-served URL in a different browser (or a new Chrome profile / incognito with third-party cookies allowed for the popup), sign in with the **same** Google account, switch to Learn mode.

Expected: the mastered count and progress bar reflect the same mastered cards seen in Step 2, without manually doing anything else.

- [ ] **Step 4: Verify a change on the second browser reaches the first**

On the second browser, master one more card. Wait ~2 seconds. Go back to the first browser, reload the page, sign in (if not already), switch to Learn mode.

Expected: the newly mastered card from the second browser shows as mastered here too.

- [ ] **Step 5: Verify offline fallback**

In the first browser (signed in), open DevTools → Network tab → set throttling to "Offline". Master another card.

Expected: no uncaught exception in the console (a caught "Could not sync progress to cloud" error is fine); the mastered count still updates in the UI immediately (state is in-memory + localStorage, independent of the network). Set throttling back to "Online", master one more card, wait ~2s, and confirm in the Firebase console that the latest mastered set now includes everything from this session.

- [ ] **Step 6: Verify sign-out fallback**

Click "Sign out". Master another card.

Expected: mastered count still updates; check DevTools → Application → Local Storage → confirm `italian-learn-progress` was updated with the new card. (No Firestore write should occur while signed out — confirmed by no new errors and no console activity referencing `db`.)

- [ ] **Step 7: Commit**

```bash
git add italian_flashcards.html
git commit -m "Sync Learn mode progress to Firestore when signed in"
```

---

### Task 5: Push and verify on GitHub Pages

**Files:** none (deployment verification only).

**Interfaces:** none — this task only pushes and verifies the already-committed changes from Tasks 2-4.

- [ ] **Step 1: Push to main**

```bash
git push
```

- [ ] **Step 2: Wait for GitHub Pages to rebuild**

GitHub Pages rebuilds automatically on push to `main` (no separate deploy command). Wait about 1 minute.

- [ ] **Step 3: Re-run the Task 4 verification steps against the live URL**

Repeat Task 4, Steps 2-6, but using `https://londonzone.github.io/italian-flashcards/italian_flashcards.html` (or the root redirect `https://londonzone.github.io/italian-flashcards/`) instead of `localhost`.

Expected: same results as Task 4 — sign-in works, progress syncs across two signed-in sessions, offline fallback doesn't throw, sign-out reverts to localStorage-only.

- [ ] **Step 4: Confirm authorized domain matches**

If sign-in fails on the live URL with an `auth/unauthorized-domain` error, go back to Firebase console → Authentication → Settings → Authorized domains and confirm `londonzone.github.io` is present (this was set up in Task 1, Step 5 — this step just double-checks it against the real deployed domain).
