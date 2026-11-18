# 2. Foundational Concepts (theory)

This file explains **every term** in the project, for someone starting from scratch. Master it and you'll be confident against any "concept" question.

---

## 2.1. PWA (Progressive Web App)

**What is a PWA?** A web app "upgraded" to behave almost like a native app: installable to the home screen, with its own icon, running fullscreen (no address bar), and **working offline**. The user doesn't need the App Store / Play Store.

**Three things that make a PWA:**
1. **HTTPS** (or localhost) — required for security.
2. **Web App Manifest** (`manifest.json`) — a JSON file declaring the app name, icons, colors, display mode, shortcuts.
3. **Service Worker** — a background script that intercepts network requests to serve from cache → offline.

**In the project:** the PWA aspect is critical because this is an emergency app — **Panic Mode must work even without network**. The manifest also declares 2 shortcuts: open straight into "Panic Mode" (`?panic=1`) and "Emergency Tools" (`?tools=1`).

---

## 2.2. Service Worker & Cache API

**What is a Service Worker (SW)?** A JavaScript file running in the "background", **separate from the web page**, acting as a **client-side network proxy**. It can intercept all of the page's network requests and decide: serve from cache or hit the network.

**The SW lifecycle:**
- **install**: installation — when you precache (preload) files into the Cache API.
- **activate**: activation — clean up old caches.
- **fetch**: every time the page requests a resource, the SW intercepts it.

**Cache API:** a request/response store right in the browser, managed by the SW. Unlike localStorage (small text strings only), the Cache API can store whole files (HTML/CSS/JS/images).

**Two caching strategies used in the project:**
- **Network-first** (for page navigation): try the network first, fall back to cache if offline. Ensures the user sees the latest version when online.
- **Stale-while-revalidate** (for the remaining assets): return the cached version immediately (fast), while quietly fetching a fresh version into cache for next time. Balances speed and freshness.

**Cache versioning:** the project uses `CACHE_VERSION = "capcuu101-v17"`. Every time a file changes, you must **bump** the version → the new SW cleans old caches and loads the new build. A key technique to avoid users being stuck on an old version.

> 💡 **Common question: "Why doesn't the SW run with `file://`?"** For security, SWs only work on `https://` or `localhost`. Opening a file directly via `file://` won't register the SW.

---

## 2.3. Web Audio API

**What is it?** A powerful API to create and process audio in the browser in real time by wiring "nodes" into an audio graph.

**How the project uses it (worth highlighting):** instead of loading `.mp3`/`.wav` files (heavy, must be cached), the project **synthesizes all sound from scratch**:
- `AudioContext` — the audio "engine".
- `OscillatorNode` — generates a sound wave (square/triangle/sawtooth) at a frequency → a "note".
- `GainNode` — controls volume over time (creating fade-in/out envelopes).
- Combining notes over time → the metronome "tick", the "ting" for correct, the "sad trombone" for losing, the win fanfare, and even 8-bit background music.

➡️ **The benefit:** no media files → an ultra-light app shell, **perfect offline operation**, no need to cache audio files.

**An important note — autoplay policy:** browsers **block audio playback** before the user interacts (tap/click). So the project has an `unlock()` function that touches the `AudioContext` on the first tap to "unlock" audio.

---

## 2.4. Web Speech API (Text-to-Speech)

**What is it?** An API that lets the browser **read text aloud** (SpeechSynthesis) — turning text into speech.

**In the project:** used for Panic Mode — reading each first-aid step aloud so the user's hands are free to act.

**Concepts to know:**
- `speechSynthesis.getVoices()` — gets the list of voices on the device. **Asynchronous**: the first call may return empty, so you listen for the `onvoiceschanged` event.
- `SpeechSynthesisUtterance` — a "spoken phrase" with text, language (`lang`), rate, pitch.
- **A real-world issue:** voices depend on the **operating system**. If the device lacks a `vi-VN` voice, the browser uses an English voice to read Vietnamese → completely wrong. The project handles this with: a voice picker in Settings, a **warning** when the Vietnamese voice is missing, and an option to **turn off read-aloud** (read the on-screen text instead).

> 💡 **A neat TTS trick in the project:** the emergency number is written as "một một năm" ("one one five") in the source data so TTS reads "115" correctly (writing "115" might be read as "one hundred fifteen"). When **displayed** on screen, the code swaps it back to "115". Similarly "năm centimet" → "5cm". A subtle detail showing you understand TTS behavior.

---

## 2.5. Other browser Web APIs

- **Vibration API** (`navigator.vibrate`): vibrates the device in a pattern. Used for the CPR beat. Mostly works on Android.
- **DeviceOrientation API**: reads the gyroscope to know how the device is tilted. Used for the Heimlich "tilt to thrust" game. iOS requires permission; devices without a sensor fall back to tapping.
- **Notifications API**: shows system notifications. Used for study reminders. *Note:* the project only reminds **when the app is reopened** (no background push) because background push needs a server + VAPID — not built.
- **Web Share API** (`navigator.share`): invokes the device's native share dialog. Used to share the result card. Falls back to copying a link if unsupported.
- **Canvas API**: draws 2D graphics via code. Used to **draw the result image card** (score/stars/badges) for sharing.
- **localStorage**: a key-value store in the browser, holds text strings, persists after closing the tab.

---

## 2.6. i18n (Internationalization)

**What is i18n?** Designing the app to support multiple languages. (Abbreviated because there are 18 letters between the "i" and "n" of "internationalization".)

**The project's home-grown i18n (no library):**
- A `LANG` variable read from `localStorage.capcuu101_lang` (default `vi`).
- A `t(v)` function: if `v` is an object `{vi, en}` → return per `LANG`; if a string → return as-is.
- **Static UI strings**: gathered in a `UI` object (key → `{vi, en}`); HTML elements get `data-i18n="key"`; an `applyLang()` function scans them all and sets `textContent`.
- **Dynamic content**: bilingual data right in the structures (level titles, lessons are `{vi,en}` objects; quizzes/panic have parallel `QUIZZES_EN`/`PANIC_EN`).
- **Long prose blocks in HTML**: use `.only-vi`/`.only-en` classes to show/hide based on `body.lang-en`.

> 💡 A great example to discuss **building a simple-but-sufficient i18n system yourself**, understanding the trade-off between "roll your own" and "use a library".

---

## 2.7. IIFE & the module pattern

**An IIFE (Immediately Invoked Function Expression)** is a function defined and **called immediately**: `(() => { ... })()`.

**Why use it?** With no module system (no build), the whole `app.js` is wrapped in an IIFE to:
- **Create a private scope**: inner variables/functions don't "leak" to global → avoiding name clashes.
- Expose only what's needed (e.g. `sound.js` exposes `window.Sfx`).
- Enable `"use strict"` for strict mode to catch errors early.

---

## 2.8. Other terms

- **App shell**: the minimal skeleton (HTML/CSS/JS) needed to display the app; precached to run offline.
- **(Client-side) router**: switches "screens" without reloading the page. Here it's the `go(id)` function toggling the `.active` class among `<section class="screen">`.
- **Splash screen**: the welcome screen when the app opens.
- **Accordion**: a UI component that expands/collapses items (used for the Handbook).
- **ARIA (Accessible Rich Internet Applications)**: HTML attributes that help screen readers understand the UI (e.g. `aria-label`, `aria-live`, `aria-hidden`).
- **`prefers-reduced-motion`**: an OS signal that the user wants reduced motion; the project respects it automatically.
- **XSS (Cross-Site Scripting)**: a flaw where malicious code is injected via user input. The project prevents it by using `textContent` (escaping) when rendering medical-card data.
- **Open Graph / Twitter Card**: `<meta>` tags that make links render nicely on social media (image + title + description).
- **Metronome**: a beat counter; here it plays 100-110/min to keep CPR on tempo.
- **Streak**: the number of consecutive days the user opens the app to review.
