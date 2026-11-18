# 4. Code Deep Dive — file by file

This file explains the **role and key functions** of each source file. Read it to stay confident when your code gets "grilled".

---

## 4.1. `app.js` — central orchestrator

The largest file, containing:

- **Imports `ROOM_META` and `ROOM_LOADERS`**; each room module loads on demand. Subsystems (i18n, store, sound, analytics, roomstate, roomquiz) are imported statically.
- **`export const ROOMS`** — the 26-room array, each with `id/icon/num/title/question/blurb/render` (bilingual).
- **`UI`** — shared UI strings (buttons, titles, search) as `{vi,en}`.
- **`SEARCH_KEYWORDS`** — bilingual search keywords per room.
- **`THEME`** — a `[accent, accent2, rgb]` color set per room.
- **WCAG helpers**: `luminance(hex)`, `contrast(l1,l2)`, `darkenForText(hex)` — auto-darken text color until contrast hits 4.5:1.
- **`applyTheme(id)`** — set the CSS color variables per room.
- **`HINTS` + `showHint(id)`** — onboarding hint shown once per room, auto-hides after 7s.
- **`openSearch()`** — the quick room finder (`/` key): filters over `ROOMS` + `SEARCH_KEYWORDS`, supports ↑↓/Enter/Esc, and **traps focus** (Tab cycles inside the dialog, returns focus to the opener when closed). Normalizes Vietnamese with `normalize("NFD")` so accent-free typing still matches.
- **`shareRoom(room)`** — Web Share API + clipboard fallback, carrying the deep-link state.
- **`toast(msg)`** — small auto-dismiss notice.
- **`buildToolbar()`** — builds the search/language/sound/present buttons.
- **`renderProgress(activeId)`** — the row of progress dots, with ARIA per room.
- **`navigate(id)`** — changes `location.hash`.
- **`route()`** — the main orchestrator (see file 03, §3.4).
- **Listeners**: `hashchange` → `route`; `langchange` → rebuild toolbar + re-render room; ←/→ navigation; `/` opens search; clicks play a soft tick.

---

## 4.2. `i18n.js` — bilingual

- `getLang()` / `setLang(l)` — read/change language (saves to localStorage, fires `"langchange"`).
- `tx(a, b)` — the translation function: takes `{vi,en}` **or** two strings.
- Wraps localStorage in `try/catch` (safe in Safari private mode).

```js
export function tx(a, b) {
  if (a && typeof a === "object") return a[lang] ?? a.vi;
  return lang === "en" ? b : a;
}
```

---

## 4.3. `store.js` — progress persistence

One localStorage key `"ai-explorer-progress"` holds `{ visited, bestScore, quizAnswered, track }`. Functions:
- `markVisited(id)` / `getVisited()` — visited rooms (returns a `Set`).
- `getBestScore()` / `setBestScore(score)` — highest quiz score (returns `true` on a new record).
- `markMicroQuiz(roomId, qIndex)` / `getMicroSolved(roomId)` / `getMicroTotal()` — micro-quizzes.
- `getTrack()` / `setTrack(track)` — selected track.
- `resetProgress()` — wipe.

Every write is wrapped in `try/catch`, every read has a safe default.

---

## 4.4. `roomstate.js` — deep-linking

- `parseHash(hash)` → `{ id, params }` (splits room and query).
- `getParams()` / `getParam(key, fallback)` — read params.
- `setParams(obj)` — write params via `history.replaceState` (skips empty/null; does **not** fire `hashchange`).
- `buildShareUrl(id, obj)` — absolute share URL.

---

## 4.5. `roomquiz.js` — the "quick check" quiz

- `BANK` — the question bank, one per room: `{ q:{vi,en}, opts:{vi:[],en:[]}, correct }`.
- `hasMicroQuiz(roomId)` — whether a room has a quiz.
- `renderMicroQuiz(container, roomId)` — inserts the quiz block at the end of a room: renders options, grades right/wrong, plays sound, highlights the correct answer, and records the point via `markMicroQuiz`. A previously-solved question is locked and pre-marked.

---

## 4.6. `tracks.js` — learning tracks

- `TRACKS` — the 3 tracks `beginner` / `full` / `dev`, each with `icon/label/desc/rooms`.
- `trackRoomIds(trackKey, allRooms)` — returns the room ids (if `rooms=null`, all of them).
- `trackLabel(trackKey)` — the translated label.

---

## 4.7. `sound.js` — sound & confetti

- Synthesizes sound with **Web Audio**: `tone(freq, dur, type, gain, when)` chains `OscillatorNode` + `GainNode` — no media files.
- `sfx` — the effect set: `click / pop / tick / success / wrong / complete`.
- `isMuted()` / `setMuted(v)` — toggle (saved to localStorage).
- `celebrate()` — draws confetti with Canvas on completion.
- `ac()` handles the **autoplay policy**: only creates/`resume()`s the `AudioContext` when needed (after a user gesture).

---

## 4.8. `pwa.js` — service worker & updates

- Registers `sw.js` on page load.
- Watches for a new worker (`updatefound` → `statechange`); once `installed` with an old controller present → calls `notifyUpdate()` to show a **"New version — Reload"** toast (bilingual via `<html lang>`).
- Clicking "Reload" → posts `"SKIP_WAITING"`; when the new worker takes over (`controllerchange`) → auto-reloads once.
- Handles `beforeinstallprompt` → shows the ⬇ install button; `appinstalled` → removes it.

---

## 4.9. `sw.js` — the service worker

- `const CACHE = "ai-explorer-v22";` — the cache version.
- `ASSETS` — the list of every file to cache offline (every `.js` module, `index.html`, `style.css`, icons…).
- `install` → precache `ASSETS` (does NOT auto `skipWaiting`).
- `message` → if it receives `"SKIP_WAITING"`, calls `skipWaiting()`.
- `activate` → delete old caches ≠ `CACHE`, then `clients.claim()`.
- `fetch` → **cache-first** for the same origin, network for the rest (e.g. Google Fonts), fallback `index.html` when offline.

---

## 4.10. A typical room file

Every room follows the same template, e.g. condensed from `neural-net.js`:

```js
import { sfx } from "../sound.js";
import { tx } from "../i18n.js";

export function roomNeuralNet(root) {
  // 1) Build the HTML content (bilingual via tx), usually .panel, .row, .takeaway
  root.innerHTML = `… <canvas id="nnCanvas"></canvas> …
    <div class="takeaway">${tx("💡 Điều cốt lõi: …", "💡 Key idea: …")}</div>`;

  // 2) Grab elements, init local state
  const canvas = root.querySelector("#nnCanvas");
  let timer = null;

  // 3) Wire up interactivity (buttons, sliders…)
  root.querySelector("#trainBtn").onclick = () => { /* … */ };

  // 4) Clean up when leaving the room (CRITICAL to avoid leaks)
  window.addEventListener("roomleave", () => { if (timer) clearInterval(timer); }, { once: true });
}
```

Common traits worth naming:
- **Every displayed string goes through `tx`** → bilingual.
- **Self-cleaning** via `roomleave` (timers, webcam, animation) — since there's no framework managing lifecycles.
- **On-device computation** (KNN, MLP, k-means, bigram…), no network → works offline.

**Webcam example (`teachable.js`)** — pure KNN: reduce the image to a 24×24 gray grid (`extractFeature`), classify by nearest neighbor (`distance`), and stop the camera on leave:
```js
window.addEventListener("roomleave", () => {
  if (predTimer) clearInterval(predTimer);
  if (stream) stream.getTracks().forEach((t) => t.stop());
}, { once: true });
```

**Neural net example (`neural-net.js`)** — a 2→H→1 MLP with hand-written `forward`/`trainStep` (manual backprop), drawing the decision boundary on canvas in real time as you drag the neuron count.

See `reasoning.js` dissected in detail in `08-code-walkthrough.md`.
