# 5. Deep Analysis of the "Point-Scoring" Features

This file helps you **tell an impressive story**. Each feature comes with: the idea, how it's done, and "why it's hard/cool".

---

## 5.1. Offline-first via Service Worker ⭐ (the most important)

**Idea:** an emergency app **must work without network** — exactly when people need it most.

**How:** the service worker precaches the whole app shell (all local files) on `install`; then every request is served from cache. Two strategies: network-first for pages, stale-while-revalidate for assets. The cache is versioned (`CACHE_VERSION`) for clean updates.

**Why it's cool:** demonstrates **offline-first** thinking and a deep understanding of the SW lifecycle (install/activate/fetch), the Cache API, and the cache-update problem. Very "battle-tested" knowledge for modern web.

**Possible follow-ups:**
- *"How do users get a new build?"* → Bump `CACHE_VERSION`; `activate` deletes old caches; `skipWaiting`+`clients.claim` apply it immediately.
- *"Why does it work offline?"* → The app shell is already cached; a failed `navigate` returns `index.html` from cache.

---

## 5.2. Synthesized audio with Web Audio ⭐

**Idea:** no audio files at all — **generate every sound from scratch** with oscillators + gain.

**How:** combine `OscillatorNode`s (square/triangle/sawtooth waves) with `GainNode`s (volume envelopes over time) to create the metronome tick, the correct ting, the lose trombone, the win fanfare, and 8-bit background music.

**Why it's cool:**
- An **ultra-light** app shell with no media to cache → perfect offline.
- Shows understanding of **digital audio** (frequency, waveforms, envelopes) and the **autoplay policy** (audio must be unlocked on the first user gesture).

---

## 5.3. Panic Mode + TTS ⭐ (the "soul" feature)

**Idea:** in an emergency people panic, hands shaking, can't read text → the app **reads each step aloud, calmly** so their hands are free.

**How:** uses the Web Speech API (SpeechSynthesis) to read each step; navigation Next/Back/Repeat; CPR specifically has a 100/min metronome + vibration to pace compressions; a call-115 button is always visible; background music auto-stops.

**Subtle details worth mentioning:**
- Handling async `getVoices()` (listening for `onvoiceschanged`).
- A **warning + voice picker** when the device lacks a `vi-VN` voice (otherwise an English voice mangles Vietnamese).
- The "one one five" trick → reads "115" correctly, displays "115".
- Multilingual: English mode switches to an `en-US` voice and says "call emergency services" instead of 115 (numbers differ by country).

**Why it's cool:** this is where **human value meets engineering** — you thought about the real experience of a panicking user, and handled many TTS edge cases.

---

## 5.4. The 5-type game engine with a `cleanup()` pattern ⭐

**Idea:** 17 levels use only 5 game mechanics (`rhythm/thrust/hose/press/quiz`), reusing a shared shell.

**The notable technical point — manual lifecycle management:** with no framework to handle teardown, each game sets `gameState.cleanup` (remove listeners, clear intervals, stop sensors); `stopGame()` calls it on leaving. Forgetting it would **leak resources** (e.g. a metronome keeps running, a sensor keeps listening).

**Why it's cool:** shows you understand **lifecycle and memory management** — something frameworks usually do for you, but here you do it yourself.

---

## 5.5. Home-grown bilingual i18n ⭐

**Idea:** full VI/EN support — UI, game content, quizzes, panic, read-aloud — with no library.

**How:** a `UI` dictionary + `t()` function + `data-i18n` attributes for static text; bilingual `{vi,en}` data for dynamic content; `.only-vi/.only-en` classes for long prose.

**Why it's cool:** shows the ability to **design a compact, sufficient system** and understand the roll-your-own vs library trade-off.

---

## 5.6. A single data source (DRY) for medical content ⭐

**Idea:** Panic Mode and the quick Handbook share the **same** `PANIC`/`PANIC_EN` data.

**Why it's cool (especially important):** for a medical app, **wrong content is the most dangerous risk**. A single data source means fixing one place is correct everywhere — the Handbook can never say one thing while Panic says another. A deliberate design decision about **content safety**.

---

## 5.7. Accessibility & Serious Mode ⭐

**Idea:** the app must work for older adults, the visually impaired, and classroom/medical settings.

**How:**
- **High contrast**, **large text**, **reduced motion** (auto-respecting the OS `prefers-reduced-motion`).
- **Serious mode**: drops the "logged out of life" humor, uses neutral wording → suitable for classrooms/medical settings.
- ARIA: `aria-label` on icon buttons, `aria-live` reading Panic steps & game instructions, `aria-hidden` on hidden screens, clear focus, the COMPRESS button playable with Space/Enter, quizzes as `<button>`.

**Why it's cool:** accessibility is often overlooked; raising it shows you design for **all kinds of users**, not just the ideal case.

---

## 5.8. Real emergency tools

- **A standalone CPR metronome**: one tap plays 110/min (tick + vibrate + count 1-30) — usable during real CPR.
- **One-tap emergency calls**: 115/113/114/111 via `tel:`.
- **A personal medical card**: blood type, allergies, conditions, contacts — stored offline, XSS-escaped, with a call-a-relative button. Shows a large, clear card for rescuers.

**Why it's cool:** turns the app from a "game" into a **real rescue tool** — showing you thought through the whole usage lifecycle (learn → when needed → use).

---

## 5.9. Sharing with Canvas + Web Share API

**Idea:** after a win, generate a nice image card (score/stars/badges) to show off.

**How:** **draw the card with Canvas** (2D graphics), then call the **Web Share API** to send it via social/messaging; fall back to copying a link if unsupported. Plus Open Graph/Twitter Card for nice link previews.

**Why it's cool:** combines two browser APIs (Canvas to draw + Web Share to send) for a natural sharing mechanism, with a fallback.

---

## 5.10. "Feature ↔ technical concept" table (memorize)

| Feature | Technical concept you demonstrate |
|---------|-----------------------------------|
| Offline-first | Service Worker (lifecycle), Cache API, two strategies, cache versioning |
| Synthesized audio | Web Audio (oscillator/gain), autoplay policy |
| Panic Mode + TTS | Web Speech API, async getVoices handling, voice edge cases |
| Game engine | manual lifecycle management, leak prevention (cleanup) |
| i18n | designing a multilingual system yourself |
| Single data source | DRY, content safety |
| Accessibility | ARIA, prefers-reduced-motion, inclusive design |
| Emergency tools | tel:, Vibration, designing for real situations |
| Sharing | Canvas, Web Share API, Open Graph, fallback |
| Tilt sensor | DeviceOrientation, iOS permission, fallback |

> 💡 **Storytelling strategy:** if asked "which feature are you most proud of?", pick **offline-first** (technical depth) or **Panic Mode + TTS** (human value + many edge cases). Both show you think about real users.
