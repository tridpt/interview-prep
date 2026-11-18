# 4. Code Deep Dive (file by file)

This file dissects each code file. Read it alongside the actual code in `cap-cuu-101/`.

---

## 4.1. `index.html` — Skeleton & static content

**Job:** holds the whole app DOM as 10 `<section class="screen">` screens, overlays, and metadata.

**Key points:**
- The `<head>` declares full **PWA + SEO + link preview**: `manifest`, `theme-color`, `apple-touch-icon`, the `apple-mobile-web-app-*` metas (to install as an app on iOS), and **Open Graph + Twitter Card** (the `og.png` 1200×630 image) so shared links render nicely.
- The `viewport` sets `user-scalable=no` — the game UI is fixed, no zoom.
- Nav buttons use a `data-goto="screen-xxx"` attribute → app.js wires a shared handler for all `[data-goto]`.
- Translatable elements get `data-i18n="key"` → `applyLang()` scans and fills the text.
- Loads `sound.js` **before** `app.js` (so `window.Sfx` is ready).

---

## 4.2. `sound.js` — The audio module (Web Audio)

**Job:** expose `window.Sfx`, synthesize all sounds, manage sound/music/vibrate settings. Wrapped in an IIFE.

**Important parts:**
- `loadSettings()/saveSettings()`: read/write `capcuu101_settings` (default `{sound:true, music:false, vibrate:true}`), wrapped in `try/catch` against corrupt JSON.
- `ensure()`: create the `AudioContext` on first use (lazy), create a `master` GainNode (overall volume). If the context is `suspended`, `resume()` it.
- `tone(freq, start, dur, type, vol)`: create **one note** — an `OscillatorNode` (the wave) + `GainNode` (volume envelope: ramp up fast then fade out with `exponentialRampToValueAtTime`).
- `play(seq)`: play a sequence of notes at time offsets; **skips if sound is off** (`!settings.sound`).
- Effects: `tick()` (metronome), `correct()` (rising ting), `wrong()`, `logout()` (the "logged out of life" sad sawtooth slide), `win()` (rising fanfare), `blip()`.
- `startMusic()/stopMusic()`: 8-bit background music via `setInterval` playing a looping melody + bass.
- `vibrate(pattern)`: calls `navigator.vibrate` if enabled and supported.
- `unlock()`: calls `ensure()` on the first tap to get past the browser's **autoplay policy**.

> 💡 **A point-scorer:** explain why **synthesizing sound** beats using files: a light app shell, perfect offline, no media to cache. And knowing about the **autoplay policy** (a user gesture is required to play sound) is a "battle-tested" detail.

---

## 4.3. `app.js` — The brain (IIFE ~2,980 lines)

Wrapped entirely in `(() => { "use strict"; ... })()`. Function groups:

### i18n
- `UI`: a `{ key: {vi, en} }` dictionary.
- `t(v)`: returns a string per `LANG` if `v` is a bilingual object.
- `applyLang()`: scans all `[data-i18n]` and sets `textContent`; toggles `body.lang-en` for `.only-vi/.only-en` blocks.
- `setLang()`: saves to localStorage + `applyLang()` + re-renders the current screen.

### Router
- `go(id)`: stops the old screen's activity, switches `.active`, updates `aria-hidden`, calls the matching render. Some references (`emergStop`, `emergRender`) are assigned dynamically so `go` can call back into submodules.

### Game engine (see file 03 §3.5 for details)
- `startGame(lv)` → first-time tutorial → `runGame(lv)` → `initRhythm/initThrust/initHose/initPress/initQuiz`.
- Each init sets `gameState.cleanup`; `stopGame()` calls it when leaving (leak prevention).
- `showResult(lv, success, stars, extra)`: a result overlay attached to **`#screen-game`** (covers the full screen, scrollable), showing stars + lesson + new badges + share/replay/list buttons.
- `startSurvival/initSurvival`: continuous questions, a countdown via `requestAnimationFrame`, one wrong = over, saves `best.survival`.

### Panic Mode & TTS
- `openPanic(key)`: pick the dataset per `LANG`, `renderPanicStep()` updates text + buttons + `speakStep()`.
- `refreshVoices()`: load `getVoices()` (async, listen for `onvoiceschanged`).
- `pickSpeakVoice()`: prefer the selected voice → the `LANG` voice → `"off"`.
- `hasViVoice()`: check to warn when the Vietnamese voice is missing.
- Panic metronome (CPR): a 600ms `setInterval`, a bouncing ring + vibration + count 1-30.
- `formatStepText()/renderPanicStep()`: swap "một một năm" → "115", "năm centimet" → "5cm" when displaying.

### Emergency tools & Handbook
- `initEmergency()` (IIFE): a standalone CPR metronome (110 BPM), the medical card (HTML-escaped against XSS, `tel:` buttons with numbers filtered to `[\d+]`), assigns `emergStop`/`emergRender`.
- `renderHandbook()`: builds an accordion from `PANIC(_EN)` by `HANDBOOK_ORDER`.

### Achievements, streak, prefs
- `loadStats()/saveStats()` (`capcuu101_stats`); `recordWin(id)` increments wins + updates the daily streak (`todayStr`, `daysBetween`); `saveBest(id, val)`.
- `checkBadges()`: returns newly unlocked badges.
- `PREFS` (`capcuu101_prefs`): contrast/large/reduce/serious → `applyPrefs()` toggles classes on `<body>`.

---

## 4.4. `sw.js` — Service Worker (offline)

**Job:** precache the app shell to run 100% offline.

```javascript
const CACHE_VERSION = "capcuu101-v17";
const APP_SHELL = ["./", "./index.html", "./style.css", "./sound.js",
                   "./app.js", "./manifest.json", ...icons, ...screenshots];
```

- **install**: `cache.add` each file via `Promise.allSettled` (one failure doesn't break install) → `skipWaiting()` so the new SW activates immediately.
- **activate**: delete any cache not starting with the current `CACHE_VERSION` → `clients.claim()` to control the page right away.
- **fetch**:
  - Only handles `GET`.
  - `request.mode === "navigate"` (page navigation) → `networkFirstPage`: try the network, cache it, fall back to `./` or `./index.html` from cache when offline.
  - Everything else → `staleWhileRevalidate`: return cache immediately, fetch in the background to update the cache.

> 💡 **Common question: "Why two different strategies?"** HTML pages need to be **fresh** (network-first) so users don't get stuck on an old version when online; assets (css/js/images) prioritize **speed** (stale-while-revalidate) since they rarely change and are versioned via `CACHE_VERSION`.

---

## 4.5. `manifest.json` — PWA declaration

- `name`, `short_name`, `description`, `icons` (192/512 + SVG, `purpose: "any maskable"`).
- `display: "standalone"` (runs like an app, no address bar), `orientation: "portrait"`.
- `theme_color`/`background_color` for the splash & status bar.
- `shortcuts`: 2 shortcuts — "Panic Mode" (`?panic=1`) and "Emergency Tools" (`?tools=1`).
- `screenshots`: images so the browser's "Install" prompt looks nicer.

---

## 4.6. The `gen_*.py` scripts (dev only)

`gen_icons.py`, `gen_screens.py`, `gen_og.py`, `gen_gif.py` use **Pillow (Python)** to generate the PNG icons, screenshots, Open Graph image, and demo GIF. They **run once** during dev and are not part of the app runtime. This is the only reason the project "touches" Python.

---

## 4.7. Summary of technical decisions

| Decision | Reason |
|----------|--------|
| Vanilla JS, no framework/library | Offline-first, absolute lightness, no supply-chain attack surface |
| Synthesize sound (Web Audio) | No media files → light app shell, perfect offline |
| Service worker + cache versioning | Runs 100% offline (vital for an emergency app) |
| Two caching strategies | Pages need freshness (network-first), assets need speed (SWR) |
| `gameState.cleanup` per game | Prevents resource leaks without a framework |
| A single data source (shared PANIC) | DRY, medical content safety |
| Home-grown i18n | Sufficient, adds no dependency |
| All data in localStorage | Private, no backend |
| Escape HTML for the medical card | Prevents XSS |
| `?panic=1` / `?tools=1` | Open emergency features directly via PWA shortcuts |
