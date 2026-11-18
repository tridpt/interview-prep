# 3. System Architecture

## 3.1. High-level diagram

```
┌──────────────────────────── BROWSER ────────────────────────────────┐
│                                                                       │
│  index.html ──▶ static DOM: 10 <section class="screen"> + data-i18n   │
│  sound.js   ──▶ window.Sfx (Web Audio: tick/correct/wrong/win/bgm+vib) │
│  app.js     ──▶ IIFE: i18n (UI dict + t()) · router go() · content     │
│                  data · game engine · Panic Mode (TTS) · tools         │
│  style.css  ──▶ dark theme, screen transitions, a11y, responsive       │
│        │              │                │                              │
│        ▼              ▼                ▼                              │
│  localStorage    Web Speech (TTS)   Web APIs:                        │
│  (stars, stats,  + voice picker     Vibration, Notifications,        │
│   prefs, medcard)                   DeviceOrientation, Web Share,    │
│                                     Canvas                            │
│                                                                       │
│                          ▼                                            │
│                   sw.js (Service Worker)                              │
│                   precache app shell → RUNS 100% OFFLINE             │
└───────────────────────────────────────────────────────────────────────┘

      ⚠️ NO outbound network requests (except "Reference Sources" links,
         which only open a new tab when the user clicks them).
```

**The core architectural trait:** this is a **fully client-side app** with **no backend**. All logic, content data, and state live in the browser. This is the biggest difference from projects with a server.

## 3.2. Layers & file responsibilities

| File | Role | Responsibility |
|------|------|----------------|
| `index.html` | Skeleton & static content | 10 screens as `<section class="screen">` + overlays; `data-i18n` for i18n |
| `sound.js` | Audio module | Exposes `window.Sfx`; Web Audio synthesis + settings; loaded **before** app.js |
| `app.js` | The brain | i18n, router, content data, game engine, Panic, tools, achievements, SW registration |
| `style.css` | UI | Theme, animation, accessibility, responsive |
| `sw.js` | Offline | Service worker: precache + caching strategies |
| `manifest.json` | PWA | App declaration, icons, shortcuts, screenshots |

> 💡 **Worth noting:** all logic is in **one `app.js` file (~2,980 lines)**, wrapped in an IIFE. This is both a strength (simple, no build, easy to cache) and a **known limitation** (large file; should be split into modules if it grows much more). Recognizing this yourself is a plus for technical maturity.

## 3.3. Startup flow (init flow) — study this

When the app opens, `app.js` runs in this order (at the end of the file):

```
1. Define UI (the i18n dictionary) + t() (the translate function).
2. Collect the .screen elements, wire events for every [data-goto] (nav buttons).
3. Define data & functions: stats/badges, LEVELS, render, game engine,
   QUIZZES(_EN), PANIC(_EN), TTS, metronome, emergency tools, handbook, settings.
4. applyLang() + applyPrefs() — apply language & preferences (contrast/large text...) to <body>.
5. Set aria-hidden on every screen except the current one (accessibility).
6. Read the query string:
      ?panic=1 → open Panic Mode directly
      ?tools=1 → open Emergency Tools directly
      default  → Home
7. maybeRemind() after 1.8s — remind to review if a streak is pending.
8. Register the service worker (on 'load').
9. hideSplash() — hide the splash screen after ~1.4s.
```

**Why read the query string?** So the 2 PWA shortcuts (`?panic=1`, `?tools=1`) open straight into the right feature — important for emergencies: opening the app jumps right into the guidance.

## 3.4. The simple router — `go(id)`

This is the client-side "router" that doesn't reload the page:

```
go(id):
  1. Stop the old screen's game/panic/metronome (avoid resource leaks).
  2. Switch the .active class to the new screen.
  3. Update aria-hidden on the screens.
  4. Call the matching render function (renderLevels / renderAchievements /
     renderHandbook / emergRender).
```

**10 screens:** `screen-home`, `screen-levels`, `screen-game`, `screen-panic`, `screen-about`, `screen-sources`, `screen-achievements`, `screen-emergency`, `screen-medcard`, `screen-handbook`. Plus overlays: `#splash`, `#tutorial`, `#settings-modal`.

## 3.5. The mini-game engine (5 game types)

All games share 3 DOM elements (`#game-stage`, `#game-instruction`, `#game-controls`) and a `gameState = { cleanup() }` variable.

**An important design pattern — `cleanup()`:** each game, on init, sets a `gameState.cleanup` function. When leaving the screen, `stopGame()` calls `cleanup()` to **tear down** (remove listeners, clear intervals, stop sensors). This is how you prevent **memory/resource leaks** without a framework managing the lifecycle.

| `type` | Function | Mechanic |
|--------|----------|----------|
| `rhythm` | `initRhythm` | Tap on beat (`interval = 60000/BPM`), tracks combo + accuracy, tick + vibrate; playable with Space/Enter |
| `thrust` | `initThrust` | DeviceOrientation (tilt) to thrust; falls back to press-hold-release without a sensor |
| `hose` | `initHose` | Pointer-drag a "hose" over the burn long enough; the burn periodically "spreads" |
| `press` | `initPress` | Step 1 drag gauze onto a circle; step 2 hold force within a moving green zone |
| `quiz` | `initQuiz` | Multiple choice, wrong answer costs 40 HP, lose at 0 HP |

Flow: `startGame(lv)` → show first-time tutorial → `runGame(lv)` dispatches by `type` → `showResult()` (score stars, lesson, new badges, share/replay buttons).

## 3.6. Data model

**Content data** (constants in `app.js`):
- `LEVELS`: an array of 17 levels — `{ id, emoji, type, char, title{}, desc{}, lesson{} }`.
- `QUIZZES` / `QUIZZES_EN`: `{ [id]: [{ q, options:[{t,ok}], wrongMsg }] }`, exactly one correct option each.
- `PANIC` / `PANIC_EN`: `{ [id]: { title, metronome?, steps:[...] } }`, 17 situations.
- `TUTORIALS`, `BADGES` (9 badges with a `test(stats, stars)` function), `HANDBOOK_ORDER`.

**User data** (localStorage):

| Key | Content |
|-----|---------|
| `capcuu101_stars` | Stars per level |
| `capcuu101_stats` | wins, totalWins, best (combo/survival), streak, badges |
| `capcuu101_settings` | sound / music / vibrate (managed by `sound.js`) |
| `capcuu101_prefs` | contrast / large / reduce / serious |
| `capcuu101_lang` | language (vi/en) |
| `capcuu101_voice` | selected voice |
| `capcuu101_tut` | which game-type tutorials have been seen |
| `capcuu101_reminder` | reminder enabled |
| `capcuu101_medcard` | personal medical card |

> 💡 **A single source of data:** the quick Handbook **reuses the exact** `PANIC`/`PANIC_EN` data from Panic Mode — no duplicated content. This is the DRY (Don't Repeat Yourself) principle, especially important for a medical app: fix one place, correct everywhere.

## 3.7. The offline layer — Service Worker (`sw.js`)

```
install  → precache APP_SHELL (all local files) into the Cache API → skipWaiting()
activate → delete any cache not matching the current CACHE_VERSION → clients.claim()
fetch    → page navigation: network-first (offline → return index from cache)
           other resources: stale-while-revalidate
```

Every asset change requires **bumping `CACHE_VERSION`** (e.g. v17 → v18) so clients get the new build. `Promise.allSettled` during precache ensures one failed file doesn't break the whole install.

## 3.8. Why is this architecture right for an emergency app?

- **Offline-first**: emergencies may lack network → the app still runs.
- **Absolutely lightweight**: no framework/library → fast load, compact cache, fewer risks.
- **Private**: no backend, all data on the user's device → no worries about leaking personal medical data.
- **Easy to deploy & robust**: static web on GitHub Pages, no server to maintain or crash.
