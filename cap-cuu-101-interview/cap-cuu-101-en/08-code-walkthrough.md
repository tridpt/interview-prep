# 8. Appendix: Code Walkthrough — reading every function

> ⚠️ This file was written after **directly reading 100% of `app.js` (2,864 lines)**, `sound.js`, `sw.js`, `index.html`, `style.css`. It is the "most thorough" part — walking through **every real function and variable** in code order. Read it with `04-code-deep-dive.md` (overview) and `app.js` open beside you.

`app.js` is **one IIFE** `(() => { "use strict"; ... })()`. Below is everything inside it, in order.

---

## PART 1 — I18N

```js
let LANG = localStorage.getItem("capcuu101_lang") || "vi";
function t(v) {
  if (v && typeof v === "object" && ("vi" in v || "en" in v)) return v[LANG] ?? v.vi;
  return v;
}
```
- `LANG`: a global read from localStorage, default `"vi"`.
- `t(v)` (translate): the **heart of i18n**. If `v` is an object with `vi`/`en` → return `v[LANG]`, falling back to `v.vi` (`??` = nullish coalescing). If `v` is a plain string → return it as-is. So `t()` works for both bilingual objects and plain strings.
- `UI`: a big `{ key: {vi, en} }` object holding **all static UI strings** (~90 keys: buttons, titles, settings labels, messages...). E.g. `play: { vi: "🎮 CHƠI & HỌC", en: "🎮 PLAY & LEARN" }`.

> 💡 Interview: "Where is all the bilingual text centralized?" → the `UI` object for static text; content (levels/quizzes/panic) is bilingual right inside the data structures.

---

## PART 2 — NAVIGATION (router)

```js
const screens = {};
document.querySelectorAll(".screen").forEach(s => (screens[s.id] = s));
let current = "screen-home";
let emergStop = null, emergRender = null;   // assigned by initEmergency
```
- `screens`: a `{ id: element }` map of every `<section class="screen">`.
- `current`: the id of the visible screen.
- `emergStop`/`emergRender`: **function references assigned dynamically** from the `initEmergency` module (since that module is its own IIFE, `go()` needs to call back into it).

**The `go(id)` function — the router:**
```js
function go(id) {
  if (!screens[id]) return;
  if (current === "screen-game") stopGame();        // tear down old game
  if (current === "screen-panic") stopPanic();       // stop TTS + metronome
  if (current === "screen-emergency" && emergStop) emergStop();  // stop the metronome
  if (id === "screen-panic" && window.Sfx) Sfx.stopMusic();      // mute music in Panic
  screens[current].classList.remove("active");
  screens[id].classList.add("active");
  screens[current].setAttribute("aria-hidden", "true");
  screens[id].removeAttribute("aria-hidden");
  current = id;
  // call the matching render:
  if (id === "screen-levels") renderLevels();
  if (id === "screen-achievements") renderAchievements();
  if (id === "screen-emergency" && emergRender) emergRender();
  if (id === "screen-handbook") renderHandbook();
}
```
➡️ **Key point:** before leaving a screen, `go()` **tears down the old screen's resources** (game/panic/metronome) — leak prevention. Then it swaps the `.active` class (CSS handles show/hide) + `aria-hidden` (accessibility) + calls the right render.

```js
document.querySelectorAll("[data-goto]").forEach(btn =>
  btn.addEventListener("click", () => go(btn.dataset.goto)));
```
- Every button with `data-goto="screen-xxx"` is auto-wired to `go()`. A "declare navigation in HTML" approach.

---

## PART 3 — STARS (`capcuu101_stars`)

```js
function loadStars()  { try { return JSON.parse(localStorage.getItem(SAVE_KEY)) || {}; } catch { return {}; } }
function saveStars(id, stars) { const all = loadStars(); if (!all[id] || all[id] < stars) all[id] = stars; localStorage.setItem(...); }
function totalStars() { return Object.values(loadStars()).reduce((a,b)=>a+b, 0); }
```
- `loadStars()`: reads `{ id: starCount }`, **wrapped in try/catch** against corrupt JSON (a pattern repeated throughout the file).
- `saveStars()`: **only overwrites if the new score is higher** (keeps the best).
- `totalStars()`: sums all stars.

---

## PART 4 — STATS, STREAK & BADGES (`capcuu101_stats`)

```js
function loadStats() {
  return Object.assign(
    { wins: {}, totalWins: 0, best: {}, streak: { count: 0, last: "" }, badges: [] },
    JSON.parse(localStorage.getItem(STATS_KEY)) || {} );
}
```
- `Object.assign(default, saved)` → guarantees all fields exist even if old data is missing some (a **lightweight migration** technique).

**Streak — the subtle logic:**
```js
function todayStr()  { ... return "YYYY-M-D"; }
function daysBetween(a, b) { ... return number of days between two strings; }
function recordWin(id) {
  const s = loadStats();
  s.wins[id] = (s.wins[id]||0)+1; s.totalWins++;
  const t = todayStr();
  if (s.streak.last !== t) {                 // not yet counted today
    const gap = s.streak.last ? daysBetween(s.streak.last, t) : 999;
    s.streak.count = gap === 1 ? s.streak.count + 1 : 1;   // consecutive day → +1, broken → reset to 1
    s.streak.last = t;
  }
  saveStats(s);
}
```
➡️ If the last play was **yesterday** (`gap === 1`) → streak +1; if further → reset to 1. Multiple plays the same day don't add (because `last === t`).

**`saveBest(id, val)`**: stores the CPR combo / survival best (only if higher).

**`BADGES` — an array of 9 badges:** each has `test(stats, stars)` returning a boolean. E.g.:
```js
{ id: "cpr_master", test: (s, st) => (st.cpr || 0) >= 3 }       // 3 stars on CPR
{ id: "all_stars",  test: (s, st) => LEVELS.every(l => (st[l.id]||0) >= 3) }  // 3 stars on all
{ id: "streak7",    test: (s) => s.streak.count >= 7 }          // 7-day streak
```

**`checkBadges()`** — returns the **newly** unlocked badges:
```js
BADGES.forEach(b => { if (!s.badges.includes(b.id) && b.test(s, st)) { s.badges.push(b.id); newly.push(t(b.name)); } });
```
Only adds badges not yet owned whose `test` passes → no duplicate alerts.

---

## PART 5 — LEVEL DATA (`LEVELS`)

An array of 17 objects, each:
```js
{ id:"cpr", emoji:"🐻", type:"rhythm", char:"🐻",
  title:{vi,en}, desc:{vi,en}, lesson:{vi,en} }
```
- `type` ∈ `rhythm | thrust | hose | press | quiz` → decides which engine runs.
- `char`: the character emoji shown in the stage.
- `lesson`: the real-life lesson shown on the result screen.
- 5 action levels (cpr/choke/infant_cpr→rhythm, burn→hose, bleed→press) + 12 `quiz` levels.

---

## PART 6 — RENDER LEVELS & ACHIEVEMENTS

**`renderLevels()`**: updates total stars, calls `renderStreakBanner()`, then:
1. Creates a **Survival** card at the top (with `role="button"`, `tabindex="0"`, listening to both click and Enter/Space → accessibility).
2. Loops `LEVELS`, each making a `.level-card` showing emoji + title + stars earned (`"★".repeat(got)+"☆".repeat(3-got)`), click → `startGame(lv)`.

**`renderStreakBanner()`**: computes whether the streak is still "alive" (`daysBetween(last, today) <= 1`), shows the right banner (keep it up / play to continue / no streak yet).

**`renderAchievements()`**: builds stats HTML (total stars, 3-star levels, streak, lives saved, best CPR combo, badge count) + a badge grid (unlocked shows the emoji, otherwise 🔒) + a share button calling `shareCard()`.

---

## PART 7 — HANDBOOK (`renderHandbook`) & `formatStepText`

```js
function formatStepText(s) {
  return s.replace("một một năm","115").replace("năm centimet","5cm")
          .replace("bốn centimet","4cm").replace("đánh quận răng","đánh răng");
}
```
➡️ **An important TTS trick:** the source data writes "một một năm" so the speech engine reads "115" correctly; when **displaying**, it swaps the number back. (The same logic is used in `renderPanicStep`.)

`renderHandbook()`: loops `HANDBOOK_ORDER` (17 keys in priority order), pulls data from `PANIC_EN`/`PANIC`, builds an **accordion** (`.hb-item`): clicking `.hb-head` toggles the `open` class + updates `aria-expanded` + `Sfx.blip()`. **Reuses the exact PANIC data** (DRY).

---

## PART 8 — GAME ENGINE (shared shell)

```js
const stage = $("game-stage"), instr = $("game-instruction"), controls = $("game-controls");
let gameState = null;
const gameScreen = $("screen-game");
function clearOverlays() { gameScreen.querySelectorAll(".result-overlay").forEach(o => o.remove()); }
```

**`startGame(lv)`**:
```js
if (window.Sfx) { Sfx.unlock(); Sfx.startMusic(); }   // unlock audio (autoplay policy) + background music
go("screen-game"); clearOverlays();
// reset title/HP/stage/controls
if (tutorialSeen(lv.type)) runGame(lv);
else showTutorial(lv, () => runGame(lv));   // first time for this game type → show tutorial
```

**`runGame(lv)`**: a `switch` on `lv.type` → calls `initRhythm/initThrust/initHose/initPress/initQuiz`.

**`stopGame()`**: `if (gameState && gameState.cleanup) gameState.cleanup(); gameState = null;` → **teardown**. Each init must set `gameState = { cleanup() {...} }`.

**`floatText(txt, color)`**: creates a `.float-text` that floats up then auto-removes after 800ms (the "PERFECT!", "-40❤️"... effects).

---

## PART 9 — SURVIVAL (`startSurvival` / `initSurvival`)

- `startSurvival()`: like startGame but enters `initSurvival()`.
- `initSurvival()`:
  - Gathers **all questions** from `QUIZZES(_EN)` into `pool`, shuffles (`sort(() => Math.random()-0.5)`).
  - `PER = 9000` ms/question. Variables: `idx, score, combo, alive`.
  - `tickTimer()`: uses `requestAnimationFrame`, computes time left, fills the `tfill` bar; **last 3 seconds tick** (`Sfx.tick()` each second); timeout → `timeout()`.
  - `ask()`: shows the question, renders the options (shuffled), each button `addEventListener(..., {once:true})`.
  - `choose()`: correct → score++, combo++, effect, next; wrong → mark red + mark the correct answer green + `end(false)`.
  - `end(cleared)`: saves `best.survival` (`saveBest`), `checkBadges()`, result overlay + share/retry/back buttons.
  - `gameState = { cleanup() { alive=false; cancelAnimationFrame(rafId); } }`.

---

## PART 10 — TUTORIAL (`TUTORIALS`, `showTutorial`)

- `TUTORIALS`: `{ gameType: { emoji, title{}, text{} } }` — guidance for all 5 game types.
- `tutorialSeen(type)` / `markTutorial(type)`: read/write `capcuu101_tut` (a `{ type: true }` object).
- `showTutorial(lv, onStart)`: shows the `#tutorial` overlay, wires the start button handler **once** (`removeEventListener` immediately on click), `markTutorial`, then calls `onStart()`.

---

## PART 11 — SHARING (`shareCard`, `buildShareImage`)

```js
function buildShareImage(bigText, subText) {
  const c = document.createElement("canvas"); c.width = c.height = 1080;
  // draw a gradient background, a red cross (arc + 2 rectangles), title, score, link
  return new Promise(res => c.toBlob(res, "image/png"));
}
async function shareCard(bigText, subText) {
  const blob = await buildShareImage(...);
  if (blob && navigator.canShare) { ... navigator.share({ files:[file], ... }); return; }
  if (navigator.share) { await navigator.share({ text, url, title }); return; }
  // fallback: clipboard
  await navigator.clipboard.writeText(text + "\n" + SHARE_URL); toast(...);
}
```
➡️ **Canvas draws the image card** → **Web Share API** (prefer sharing with the image file; else share text; finally fall back to copying a link). It catches `AbortError` (user cancelled the dialog) to avoid an error.

`toast(msg)`: shows a floating message for 2.2s.

---

## PART 12 — `showResult(lv, success, stars, extra)`

- On win: `saveStars` + `recordWin`; plays `Sfx.win()` (lose: `Sfx.logout()`) + vibration.
- `checkBadges()` → shows a notice if a new badge unlocked.
- **Serious mode (`PREFS.serious`)**: swaps the emoji 💀→⚠️ and changes "LOGGED OUT OF LIFE" → "Not quite — try again".
- Shows the lesson (`t(lv.lesson)`) + Share/Retry/List buttons. The overlay attaches to `#screen-game` (full-screen).

---

## PART 13 — THE 5 GAME ENGINES (detailed)

### `initRhythm(lv)` — rhythm CPR
- `TARGET=20` good beats, `BPM=110`, `interval=60000/BPM`, `tolerance=230` ms.
- `setInterval(tick, interval)`: each beat plays `Sfx.tick()` + vibrate + bounces the `ring`.
- `onTap()`: computes `phase = now - last`, `diff = min(phase, interval-phase)`. If `diff <= tolerance` → correct (combo++, "PERFECT!"); else → "Too fast/Too slow". Reaching `TARGET` → stars by accuracy `acc` (>0.9→3, >0.7→2, else 1), `saveBest(lv.id, bestCombo)`, `showResult`.
- Playable with **Space/Enter** (accessibility).
- `cleanup() { clearInterval(beatTimer); }`.

### `initThrust(lv)` — Heimlich (tilt sensor)
- `NEED=6` strong thrusts. `doThrust(strong)`: strong enough → thrusts++, reaching NEED → `win()`.
- **`useTiltMode()`**: listens to `deviceorientation`, reads `e.beta`, normalizes to 0-100; above 65 (and `armed`) → one thrust; below 40 → `armed=true` again (prevents continuous counting).
- **`useTapMode()`**: fallback press-hold-release; held >250ms = strong thrust.
- **Mode selection**: iOS (has `DOE.requestPermission`) → a permission button; plain DOE → tiltMode + a **2.5s probe**, if no sensor data → auto-fallback to tap; no DOE → tap.
- `cleanup()`: removes the `deviceorientation` listener.

### `initHose(lv)` — burn cooling
- `NEED=6500` ms cooling. `place()`: places the wound randomly in the stage.
- Pointer events (`pointerdown/move/up/leave`) move the hose; `pointerInside` checks whether it's over the wound (comparing coords with `getBoundingClientRect`).
- `setInterval(loop, 60)`: if spraying correctly → `cooled += dt`, and **the burn periodically "spreads"** (`sinceFlare > 2000` → `place()` again + `Sfx.wrong()`); if off the wound → `cooled` decays (heats up). Reaching `NEED` → win 3 stars.
- `cleanup()`: clearInterval + removes all pointer listeners.

### `initPress(lv)` — bleeding control (2 steps)
- **Step 1 — drag gauze**: pointer drag the gauze; `gup()` computes the distance between the gauze center and the wound center (`Math.hypot`), `<55px` → placed correctly → `startHold()`.
- **Step 2 — hold force**: `startHold()` builds a force bar; `setInterval(60ms)`:
  - **Oscillating green zone**: `zLo = 52 + Math.sin(drift*0.9)*16` (36%-68%), `zHi = zLo+26`.
  - `force += (holding ? 0.18 : -0.22) * dt/16` (holding increases, releasing decreases).
  - If `force` is in [zLo, zHi] → `inZone += dt`; reaching `NEED_ZONE=3500` ms → win.
- `cleanup()`: clearInterval + removes listeners.

### `initQuiz(lv)` — multiple choice
- `qs = QUIZZES_EN[lv.id] || QUIZZES[lv.id]` (by language). `hp=100`.
- `render()`: out of questions → stars by remaining HP (100→3, ≥60→2, else 1) → win. Still questions → show question + **shuffled** options, each button `{once:true}`.
- `pick()`: correct → effect + next; wrong → mark red, **mark the correct answer green** (`item.options.find(o=>o.ok).t`), `hp -= 40`, if `hp<=0` → lose (show `wrongMsg`), else next.
- `cleanup() {}` (quiz has no timer/listener to clean).

---

## PART 14 — CONTENT DATA: `QUIZZES`, `QUIZZES_EN`, `PANIC`, `PANIC_EN`

- `QUIZZES`/`QUIZZES_EN`: `{ id: [ { q, options:[{t, ok}], wrongMsg } ] }`. Each question has **exactly one correct** option `ok:true`. (This is the **most dangerous data if wrong** for a medical app — see the testing section in file 06.)
- `PANIC`/`PANIC_EN`: `{ id: { title, metronome?, steps:[...] } }`. 17 situations. CPR/infant_cpr have `metronome:true`. Steps write "một một năm" so TTS reads correctly.

---

## PART 15 — PANIC MODE & TTS

**Event wiring:** every `.panic-card` click → `openPanic(c.dataset.emergency)`; the next/prev/replay/stop-voice/back-pick buttons.

**`openPanic(key)`**: picks the dataset by `LANG`, hides the picker and shows steps, if `data.metronome` → shows the ring + `startMetronome()`, then `renderPanicStep()`.

**`renderPanicStep()`**: shows `"{n}. " + text` (swapping "một một năm"→"115"...), enables/disables prev, changes next's label to "Done" on the last step, then `speakStep()`.

**`panicStep(dir)`**: forward/back; past the last step → go Home.

**TTS:**
```js
let allVoices = [];
let chosenVoiceURI = localStorage.getItem("capcuu101_voice") || "";  // "" auto, "off" off
function refreshVoices() { allVoices = speechSynthesis.getVoices() || []; if (onVoicesReady) onVoicesReady(); }
speechSynthesis.onvoiceschanged = refreshVoices;   // getVoices() is async!
function hasViVoice() { return voicesForLang("vi").length > 0; }
function pickSpeakVoice() {
  if (chosenVoiceURI === "off") return "off";
  if (chosenVoiceURI) { find voice by URI; }
  return voicesForLang(LANG==="en"?"en":"vi")[0] || null;   // auto
}
function speakStep() {
  speechSynthesis.cancel();              // cancel the current utterance
  const v = pickSpeakVoice(); if (v === "off") return;
  const u = new SpeechSynthesisUtterance(raw);
  if (v) { u.voice=v; u.lang=v.lang; } else { u.lang = LANG==="en"?"en-US":"vi-VN"; }
  u.rate = 0.95; speechSynthesis.speak(u);
}
```
➡️ Full handling: getVoices is **async**, prefer the chosen voice → the language voice → off; `speechSynthesis.cancel()` before speaking a new line (avoids overlapping speech).

**Panic metronome:** `startMetronome()` — `setInterval(600ms)` = 100 beats/min, counts 1-30, bounces the ring + vibrates 40ms. `stopMetronome()` clearInterval.

**`stopPanic()`**: stops the metronome + `speechSynthesis.cancel()`.

---

## PART 16 — REMINDERS (Notifications)

- `reminderOn()`: reads `capcuu101_reminder === "1"`.
- `setReminder(on)`: requests `Notification.requestPermission()`; granted → save + fire a confirmation notification.
- `fireReminder(body)`: prefers `serviceWorker.ready.showNotification` (nicer), falls back to `new Notification`.
- `maybeRemind()`: on app open, if reminders on + a streak exists + not played today → fire a reminder. **Only reminds on app open** (no background push).

---

## PART 17 — PREFERENCES (a11y + serious)

```js
let PREFS = loadPrefs();   // { contrast, large, reduce, serious }
function applyPrefs() {
  document.body.classList.toggle("a11y-contrast", PREFS.contrast);
  document.body.classList.toggle("a11y-large", PREFS.large);
  document.body.classList.toggle("a11y-reduce", PREFS.reduce);
  document.body.classList.toggle("mode-serious", PREFS.serious);
}
```
➡️ Toggles classes on `<body>`, CSS handles the rest (a tidy **"state in JS, style in CSS"** pattern).

---

## PART 18 — SETTINGS MODAL (IIFE `initSettings`)

- Syncs checkboxes with `Sfx.settings` (sound/music/vibrate) and `reminderOn()`.
- Each checkbox change → calls `Sfx.setSound/setMusic/setVibrate` (plays a test tone/vibration to confirm).
- A `prefMap` links the 4 a11y checkboxes to `PREFS` → `savePrefs()` + `applyPrefs()`.
- **Voice picker**: `populateVoiceSelect()` fills "Auto" + "Off" + Vietnamese voices first + the rest; `updateVoiceWarn()` shows a warning if the Vietnamese voice is missing; the test button → `speakSample()`. `onVoicesReady = populateVoiceSelect` so it re-runs when voices load (async).

---

## PART 19 — EMERGENCY TOOLKIT (IIFE `initEmergency`)

- **Standalone CPR metronome** (110 BPM): `startMetro/stopMetro` toggle a `setInterval`, count 1-30, `Sfx.tick()`+vibrate. Assigns `emergStop = stopMetro` so `go()` can stop it.
- **Medical card** (`capcuu101_medcard`):
  - `esc(s)`: **anti-XSS** — `div.textContent = s; return div.innerHTML;` (escapes special chars).
  - `renderMC()`: builds the card; the phone number → `tel:` after **filtering to `[\d+]`** (`replace(/[^\d+]/g,"")`).
  - `form submit`: `preventDefault`, gathers `.trim()`ed values, saves to localStorage, `toast`. Assigns `emergRender = renderMC`.

---

## PART 20 — APPLY LANGUAGE & INIT

**`applyLang()`**:
- Toggles `body.lang-en`, sets `documentElement.lang`.
- Scans `[data-i18n]` and sets `textContent` from `UI`.
- Updates `.panic-card` labels from each PANIC datum's `title`.
- Updates the `.lang-opt` button states (`active` + `aria-pressed`).
- Sets `aria-label` on icon buttons (back/stop-voice/settings).

**`setLang(lang)`**: saves to localStorage + `applyLang()` + re-renders the current screen.

**The INIT block (runs immediately):**
```js
applyLang(); applyPrefs();
// aria-hidden on every screen except current
hideSplash();   // hide the splash after a minimum of 1.4s
// read query: ?panic=1 → Panic; ?tools=1 → Tools; else Home
setTimeout(maybeRemind, 1800);
// register the service worker on 'load'
if ("serviceWorker" in navigator) window.addEventListener("load", () => navigator.serviceWorker.register("sw.js").catch(...));
```

---

## 🎯 Quick "function ↔ purpose" reference (all of app.js)

| Group | Main functions |
|-------|----------------|
| i18n | `t`, `UI`, `applyLang`, `setLang` |
| Router | `go`, `screens`, `current` |
| Storage | `loadStars/saveStars/totalStars`, `loadStats/saveStats`, `recordWin`, `saveBest` |
| Streak | `todayStr`, `daysBetween`, `recordWin` |
| Badges | `BADGES`, `checkBadges` |
| Data | `LEVELS`, `QUIZZES(_EN)`, `PANIC(_EN)`, `TUTORIALS`, `HANDBOOK_ORDER` |
| Render | `renderLevels`, `renderStreakBanner`, `renderAchievements`, `renderHandbook` |
| Engine | `startGame`, `runGame`, `stopGame`, `initRhythm/Thrust/Hose/Press/Quiz`, `showResult`, `floatText` |
| Survival | `startSurvival`, `initSurvival` |
| Tutorial | `tutorialSeen`, `markTutorial`, `showTutorial` |
| Sharing | `buildShareImage`, `shareCard`, `toast` |
| Panic | `openPanic`, `renderPanicStep`, `panicStep`, `formatStepText`, `stopPanic` |
| TTS | `refreshVoices`, `voicesForLang`, `hasViVoice`, `pickSpeakVoice`, `speakStep`, `speakSample` |
| Metronome | `startMetronome`, `stopMetronome` (Panic) + `initEmergency` (standalone) |
| Reminders | `reminderOn`, `setReminder`, `fireReminder`, `maybeRemind` |
| Prefs | `loadPrefs`, `savePrefs`, `applyPrefs` |
| Modal | `initSettings` (IIFE) |
| Toolkit | `initEmergency` (IIFE): metronome + medical card (`esc`, `renderMC`) |
| Startup | the INIT block + SW registration |

> 💡 Master this table + read through file 08 once, and you can **point at any function in `app.js` and explain it** — that's the "most thorough" level for an interview.
