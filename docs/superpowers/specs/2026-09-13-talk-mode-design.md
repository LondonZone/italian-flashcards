# Talk Mode — Design Spec (First Build)

## Context

Adding a 4th practice mode, "Talk," for spoken conversation practice with
real-time correction, modeled on the app "Lua." User speaks Italian, gets
transcribed, gets a natural Italian reply plus corrections, reply is spoken
back out loud.

The original build brief covered a larger surface (hardcoded scenario seed
list, growing it into a Firestore-backed library, and a self-serve
`generateScenario` authoring function). **This spec scopes only the first
build**: audio capture, the two core Cloud Functions, the hardcoded scenario
array, the chat UI, and the mistake tracker + conversation history. The
Firestore-backed scenario library and `generateScenario` are deferred to a
follow-on spec once the first build is in use.

## Scope decisions made during brainstorming

These deviate from (or resolve ambiguity in) the original brief — noted
explicitly so they aren't mistaken for oversights:

- **AI provider: OpenAI for both.** Whisper (`whisper-1`) for transcription,
  GPT-4o-mini for chatReply. Single API key (`OPENAI_API_KEY`), cheapest
  combined option at this usage volume (a personal practice tool, not a
  product with many users).
- **Conversation history IS persisted per topic in Firestore.** The original
  brief said "full raw transcripts are explicitly out of scope — David only
  wants the mistake data, not a conversation log." That's reversed here: the
  user explicitly asked for a profile that resumes conversations per
  scenario. Mistake *events* are still tracked separately as their own
  aggregable records (per the original brief), but full turn-by-turn history
  is also now stored.
- **Talk mode requires sign-in.** No anonymous/local-only fallback path.
  Progress and history are tied to the Firebase Google account already used
  for cross-device sync elsewhere in the app.
- **Resuming a scenario loads its prior history** into the live chat (not
  just a passive log) and feeds it as context to `chatReply`. A "Start Over"
  button clears it.

## Architecture

### New Firebase infra

This repo currently has no `functions/` directory or `firebase.json` — Cloud
Functions don't exist yet, only the client SDK (Firebase Auth + Firestore)
loaded via CDN `<script>` tags directly in `italian_flashcards.html`. This
build adds:

- `functions/` (Node, Firebase Functions v2, callable functions) with
  `index.js` exporting `transcribeAudio` and `chatReply`.
- `firebase.json` + `.firebaserc` for deploy config.
- `OPENAI_API_KEY` set via `firebase functions:secrets:set` — never present
  in client-side JS.
- **Manual one-time prerequisite:** the Firebase project must be on the
  Blaze (pay-as-you-go) plan to allow outbound network calls from Cloud
  Functions to OpenAI. This is a Firebase Console action, not something
  this build can automate.

### Audio capture (client)

`MediaRecorder`, not the browser's built-in `SpeechRecognition` API (the
latter is unreliable/unsupported on iOS Safari, and this needs to work on
David's phone). Feature-detect supported mime type: `audio/mp4` on
Safari/iOS, `audio/webm` elsewhere. Record on mic-tap, stop on second tap,
send the resulting blob (base64-encoded) to `transcribeAudio`.

### `transcribeAudio` (Cloud Function, callable)

Input: `{ audioBase64, mimeType }`. Forwards to OpenAI's Whisper API
(`whisper-1`, `language: "it"`). Returns `{ transcript }`. If Whisper returns
an empty/near-empty transcript, the client shows "Didn't catch that, try
again" instead of proceeding to `chatReply`.

### `chatReply` (Cloud Function, callable)

Input: `{ history, transcript, topicId }`. The scenario's
`systemPromptAddition` is looked up **server-side** from the hardcoded
scenario array (not trusted from the client) to prevent arbitrary persona
injection. Builds a system prompt: A1-level Italian tutor, in-character per
scenario, explicitly watching for David's two known recurring errors
(importing English word order, using infinitives where a conjugated verb is
needed), tagging every correction with a category from the fixed taxonomy.

Calls GPT-4o-mini with structured output (`response_format: json_schema`)
enforcing:

```json
{
  "reply_italian": "...",
  "corrections": [
    { "mistake": "...", "correction": "...", "explanation": "...", "category": "word_order" }
  ]
}
```

Category taxonomy: `word_order`, `infinitive_vs_conjugated`,
`gender_agreement`, `verb_conjugation`, `vocabulary`, `other`.

### Scenario Library (seed set, hardcoded)

Plain JS array in the client, no backend:

```js
const conversationTopics = [
  { id: "cafe-order", title: "Ordering at a Café", icon: "☕",
    systemPromptAddition: "You are a barista in a small café in Calabria. Stay in character. Keep vocabulary at A1 level." },
  { id: "small-talk", title: "Small Talk with a Stranger", icon: "💬",
    systemPromptAddition: "You are a friendly stranger making casual conversation — weather, weekend plans, that kind of thing." },
  { id: "directions", title: "Asking for Directions", icon: "🗺️",
    systemPromptAddition: "You are a local giving directions in an Italian town." },
  { id: "texting-friend", title: "Texting a Friend", icon: "📱",
    systemPromptAddition: "You are a close friend catching up casually, like a text exchange, not formal speech." }
];
```

The same array (ids, titles, prompt additions) must be mirrored server-side
in `functions/index.js` for the `chatReply` lookup — no shared module system
between the static client HTML and the Functions Node project, so keep them
in sync manually until the Firestore migration (follow-on spec) removes the
duplication.

### Data model (Firestore)

```
users/{uid}/conversations/{topicId}
  turns: [{ role: 'user'|'assistant', text, corrections?, timestamp }]
  updatedAt

users/{uid}/mistakes/{mistakeId}
  category, original, corrected, explanation, topicId, timestamp
```

`conversations/{topicId}` is a single doc per scenario (not one doc per
turn) — simplest to resume/overwrite, fine at this volume (one user, a few
dozen turns per scenario). Written after each completed turn, using the
existing debounced-save pattern already in the codebase
(`firestoreSaveTimer`). `mistakes` gets one doc per correction, written in
the same batch as the conversation update — only after both `transcribeAudio`
and `chatReply` succeed, so nothing partial is ever written.

### Frontend — "Talk" tab

Added as a 4th `mode-btn` alongside Flashcards / Fill in Blank / Learn,
following the existing view-switching pattern (`#modeCardsBtn` etc. at
`italian_flashcards.html:411-413`).

- **Signed-out state:** if `!hasFirebaseUser()`, show a "Sign in to use Talk
  mode" prompt (reuses the existing Google sign-in button/flow) instead of
  the topic picker.
- **Topic picker:** grid of the 4 seed scenarios (icon + title). Selecting
  one loads `users/{uid}/conversations/{topicId}` if present and renders
  prior turns as chat bubbles (resume); otherwise starts empty.
- **Mic button:** tap to start recording, tap again to stop → send blob to
  `transcribeAudio` → render the transcript as a user chat bubble
  optimistically.
- **Chat flow:** on transcript received, call `chatReply` with
  `{history, transcript, topicId}` → render `reply_italian` as an assistant
  bubble, corrections listed underneath the *user's* turn (not the
  assistant's) → pipe `reply_italian` through the existing TTS flow
  (Cloudflare Worker primary, `speechSynthesis` fallback — see
  [[tts-cloudflare-worker]]) → reuse the existing tap-word-for-translation
  behavior on both the transcript and the reply text.
- **"Start Over" button:** clears `conversations/{topicId}` in Firestore and
  resets in-memory history for that scenario.

### Mistake Tracker — "Mistakes" tab

5th top-level tab. Queries `users/{uid}/mistakes`, aggregates client-side by
`category`, renders as a sorted list ("infinitive_vs_conjugated: 12 times
this month", ...) most-frequent first. No date filtering or charts in this
first build — just the aggregate count list.

### Error handling

- Mic permission denied → inline message, no crash.
- `transcribeAudio`/`chatReply` failure (network, API error, quota) →
  inline error, user can retry that turn; nothing partially written to
  Firestore.
- Empty/silent transcript → "Didn't catch that, try again," skip
  `chatReply`.
- TTS playback failure on the AI reply → falls through to the existing
  browser-voice fallback already in place; no new logic needed.

### Testing

No existing test suite in this repo (static HTML, no build step) —
verification is manual: deploy functions to the Firebase project, exercise
the full loop (record → transcribe → reply → speak → correction display →
Firestore write → reload/resume) in a browser, and specifically on an
iPhone for the `MediaRecorder` mime-type path, since that's the whole reason
`SpeechRecognition` was avoided.

## Explicitly out of scope (this spec)

- Firestore-backed scenario library (`scenarios/{id}` collection) — seed
  array stays hardcoded (client + server-mirrored) for now.
- Self-serve `generateScenario` Cloud Function.
- Tagging scenarios with card categories to cross-link Talk mode and
  Flashcards mode.
- Date filtering, charts, or trends in the Mistake Tracker beyond a sorted
  count list.
