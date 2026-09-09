# Pronunciation audio via Cloudflare Worker — design

## Problem

The flashcard app spoke Italian by loading an unofficial Google endpoint
(`translate.google.com/translate_tts`) into an `Audio` element. Google now
returns **HTTP 503** for browser-origin requests to that endpoint (it requires a
`Referer: translate.google.com` header and a signed token a web page cannot
provide). The old code's `audio.play()` promise also hung without rejecting, so
the `speechSynthesis` fallback never ran — the 🔊 button produced silence.

An interim fix (commit `ccbfe6b`) removed the dead Google path and made the
browser's `speechSynthesis` the primary voice. That works on most phones and
browsers but is robotic and is completely silent on some macOS Chrome installs
(OS-level speech engine bug).

## Solution

A Cloudflare Worker proxies the same Google Translate TTS request **server-side**,
where the `Referer` header can be set. The app calls the Worker instead of Google
directly, and keeps the browser voice as an automatic fallback.

### Worker (`italian-tts`)

- Deployed at `https://italian-tts.theathleticphysio.workers.dev`.
- `GET /?q=<text>&tl=it` — `tl` optional, defaults to `it`.
- Validates `q` is present and ≤ 200 chars (Google's limit); 400 otherwise.
- Fetches `translate.google.com/translate_tts?ie=UTF-8&client=tw-ob&tl=…&q=…`
  with a `Referer: https://translate.google.com/` header and a desktop
  `User-Agent`.
- On success: streams the MP3 back with `Content-Type: audio/mpeg` and
  `Cache-Control: public, max-age=604800`. `cf: { cacheEverything: true }` lets
  Cloudflare's edge cache repeated words.
- On upstream failure or a non-audio response: returns HTTP 502.
- ~35 lines, no dependencies, no secrets, no build step.

### App change (`italian_flashcards.html`)

- New constant `TTS_WORKER` holds the Worker base URL.
- `speakItalian(text)`:
  1. Clean the text (strip parenthetical hints and everything after `/`).
  2. If no `TTS_WORKER` or the cleaned text is > 200 chars → use the browser
     voice directly.
  3. Otherwise `new Audio(TTS_WORKER + '/?q=' + encodeURIComponent(clean))`.
  4. A 2 s timeout plus `error` / `play()`-rejection handlers call
     `speakWithBrowserVoice(clean)` if the Worker doesn't start playing.
  5. A `playing` event cancels the timeout — success, no fallback.
- The former `speakItalian` body became `speakWithBrowserVoice`. All the
  `speechSynthesis` workarounds (voice selection, deferred `cancel()`+`speak()`,
  `resume()` watchdog, one-shot retry) are unchanged — they are now the fallback.

## Data flow

```
🔊 tap
  → speakItalian(text)
     → Audio(worker URL).play()
          ├─ playing            → Google voice (via Worker + edge cache)
          └─ error / timeout    → speakWithBrowserVoice()
                                     → speechSynthesis (offline, robotic)
                                        └─ unsupported → alert()
```

## Error handling / rollback

- Worker down, network offline, or Google blocks Cloudflare's IPs → the app
  falls back to the browser voice automatically. No user-visible breakage.
- If Google blocks the Worker long-term, the next step is to switch the Worker's
  upstream to the official Google Cloud TTS API (API key stored as a Worker
  secret). No app change needed beyond that.

## Testing

- Worker verified returning `200` / `audio/mpeg` for `ciao`, `sono`, an
  accented phrase (`Lei è simpatica`), and a ~40-char sentence with an
  apostrophe.
- App verified building the correct Worker URL and falling back to the browser
  voice when `Audio.play()` fails.
- Manual check outstanding: confirm audible playback from the deployed page on a
  phone (browser automation cannot satisfy the audio user-gesture requirement).
