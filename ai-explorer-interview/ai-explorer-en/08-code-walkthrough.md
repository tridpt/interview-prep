# 8. Code Walkthrough — the real run

This appendix walks through the **real run** of the core parts, so you understand the source cold when "grilled deeply". Three parts: (1) startup + `route()`, (2) a typical room file, (3) the Reasoning room dissected.

---

## 8.1. Startup → rendering the first room

1. `index.html` loads `<script type="module" src="app.js">`. Being a module, all `import`s are resolved first.
2. At the end of `app.js`:
   ```js
   buildToolbar();   // build the search/language/sound/present buttons
   initAnalytics();  // anonymous stats (optional)
   route();          // render for the current hash
   ```
3. `route()` reads `currentRoute()` (`parseHash().id`). No hash in the URL → `id = "home"`.
4. For home: `renderProgress(null)` + `renderHome(app, ROOMS, navigate)` → draws the hero, track picker, room-card grid.
5. In parallel, `pwa.js` registers `sw.js`. First visit: the SW `install`s and precaches `ASSETS`. Later: served from cache → opens instantly, even offline.

## 8.2. `route()` when opening a room (e.g. `#/reasoning`)

```js
function route() {
  const id = currentRoute();
  window.dispatchEvent(new CustomEvent("roomleave")); // (1) clean the old room
  app.innerHTML = "";
  window.scrollTo({ top: 0, behavior: "smooth" });
  applyTheme(id === "home" ? "home" : id);            // (2) switch theme color
  const room0 = ROOMS.find((r) => r.id === id);
  trackView(id === "home" ? "/" : "/" + id, room0 ? tx(room0.title) : "AI Explorer");
  app.classList.remove("enter"); void app.offsetWidth; app.classList.add("enter"); // (3) animation

  if (id === "home") { renderProgress(null); renderHome(app, ROOMS, navigate); return; }

  const idx = ROOMS.findIndex((r) => r.id === id);
  if (idx === -1) { navigate("home"); return; }        // (4) unknown hash → home

  const room = ROOMS[idx];
  visited.add(room.id); markVisited(room.id);           // (5) mark visited
  renderProgress(room.id);

  // (6) shared frame: title + share button
  const head = document.createElement("div"); head.className = "room-head";
  head.innerHTML = `… ${tx(room.question)} … <h2>${tx(room.title)}</h2> …
    <button id="shareRoomBtn">${tx(UI.shareRoom)}</button>`;
  app.appendChild(head);
  head.querySelector("#shareRoomBtn").onclick = () => shareRoom(room);

  const body = document.createElement("div");
  app.appendChild(body);
  const renderRoom = await room.render();                // (7) load room module on demand
  if (generation !== routeGeneration || currentRoute() !== room.id) return;
  renderRoom(body);
  if (room.id !== "summary" && hasMicroQuiz(room.id)) renderMicroQuiz(body, room.id); // (8) micro-quiz
  showHint(room.id);                                     // (9) onboarding hint

  // (10) Prev / Next buttons from ROOMS[idx-1] / ROOMS[idx+1]
  const prev = ROOMS[idx - 1], next = ROOMS[idx + 1];
  // …build buttons…
}
```

Key things to grasp:
- **`roomleave`** fires *before* clearing the DOM → the old room cleans up timers/webcam in time.
- The frame (head, quiz, nav) is handled by `app.js` → rooms only write content.
- A hash matching no room → auto-return to home (dead-link protection).

## 8.3. A typical room file (the template)

Every room follows the same shape (example: `neural-net.js`):

```js
import { sfx } from "../sound.js";
import { tx } from "../i18n.js";

export function roomNeuralNet(root) {
  root.innerHTML = `… <canvas id="nnCanvas"></canvas> …
    <div class="takeaway">${tx("💡 Điều cốt lõi: …", "💡 Key idea: …")}</div>`;

  const canvas = root.querySelector("#nnCanvas");
  let net = initNet(6), timer = null, epochs = 0;

  root.querySelector("#trainBtn").onclick = (e) => {
    if (timer) { clearInterval(timer); timer = null; return; }
    timer = setInterval(() => {
      for (let i = 0; i < 3; i++) { trainStep(net, data, 0.1); epochs++; }
      refresh(); // redraw boundary + update accuracy
    }, 40);
  };

  window.addEventListener("roomleave", () => { if (timer) clearInterval(timer); }, { once: true });
}
```

Three things every room does:
1. **Build HTML via `tx`** (bilingual), usually with `.panel`, `.row`, `.takeaway`.
2. **Wire up interactivity** (buttons, sliders, canvas).
3. **Clean up** on `roomleave` (timers/animation/webcam) — mandatory since there's no framework.

`neural-net.js` hand-writes forward propagation (`forward`) and manual backprop (`trainStep`) for a 2→H→1 MLP, then paints the prediction region on canvas in 7px cells — a visual "the AI is learning".

## 8.4. The Reasoning room (`reasoning.js`) — dissected

This is the room to memorize since it's the timely highlight.

### Data structure
```js
const PROBLEMS = [
  { id: "strawberry",
    q:    { vi: "…", en: "How many 'r' letters are in 'strawberry'?" },
    fast: { vi: "2", en: "2" },              // gut answer (WRONG)
    fastNote: { vi: "…", en: "A quick gut guess…" },
    steps: [ /* reasoning steps, bilingual */ ],
    answer: { vi: "3", en: "3" },            // correct answer
    fastTok: 1, thinkTok: 48,                // illustrative token counts
  },
  // penraser (a CRT-style pen–eraser problem), weekday (day-of-week calc)…
];
```
Each problem is **deterministic**: the wrong answer, step chain, correct answer, and token counts are all fixed — so the demo is always consistent and needs no real model call.

### Restoring state from the deep-link
```js
const startP = parseInt(getParam("p", ""), 10);
let pIdx = Number.isInteger(startP) && PROBLEMS[startP] ? startP : 0;
```
Reads the `p` param from the URL (`#/reasoning?p=2`) to open the exact shared problem; invalid/absent → problem 0.

### Respecting reduced motion
```js
const reduce = window.matchMedia?.("(prefers-reduced-motion: reduce)").matches;
const stepDelay = reduce ? 0 : 700;
```
If the user prefers reduced motion, the steps appear *instantly* instead of trickling in.

### "Answer instantly" — impulsive, usually wrong
```js
fastBtn.onclick = () => {
  const p = PROBLEMS[pIdx];
  fastBtn.disabled = true;
  fastOut.innerHTML = `
    <div class="rs-answer bad">${tx(UI.answerLabel)} <b>${tx(p.fast)}</b></div>
    <p class="muted rs-note">${tx(p.fastNote)}</p>
    ${tokBadge(p.fastTok)}`;   // ≈ 1–3 token badge
  sfx.wrong();
};
```

### "Think step by step" — builds up then reaches the right answer
```js
thinkBtn.onclick = () => {
  const p = PROBLEMS[pIdx];
  clearTimers();
  thinkOut.innerHTML = `<ol class="rs-steps"></ol>`;
  const list = thinkOut.querySelector(".rs-steps");

  p.steps.forEach((s, si) => {
    const reveal = () => { /* append <li> step si */ sfx.tick(); };
    if (stepDelay === 0) reveal();
    else timers.push(setTimeout(reveal, stepDelay * (si + 1)));  // trickle each step
  });

  const finish = () => { /* append correct answer .good + tokBadge(p.thinkTok) */ sfx.success(); };
  if (stepDelay === 0) finish();
  else timers.push(setTimeout(finish, stepDelay * (p.steps.length + 1)));
};
```

### Selecting a problem & saving the deep-link
```js
function selectProblem(i) {
  pIdx = i;
  setParams({ p: i || null });   // write ?p=i to the URL (replaceState); drop param for problem 0
  clearTimers();                 // clear old-problem timers
  // reset the two output columns to a "waiting" state…
}
```

### Cleaning up on leave
```js
const timers = [];
const clearTimers = () => { timers.forEach(clearTimeout); timers.length = 0; };
window.addEventListener("roomleave", clearTimers, { once: true });
```

### The core idea the room conveys (`.takeaway` block)
> An AI isn't "smarter" when it thinks step by step — it just gets **more room to compute**. Each written step becomes context for the next, so the solution builds up instead of being guessed in one shot. The cost is being **slower and using far more tokens**, so we only need to "think hard" on tough problems — exactly what reasoning models do automatically.

> 💡 When telling this room in an interview, hit three points: (1) the **mechanism** (each step is context for the next), (2) the **trade-off** (tokens/latency ⟷ accuracy, shown via token badges), (3) the **difference** between CoT (prompting technique) and a reasoning model (a model trained to think on its own, like o1/DeepSeek-R1).

---

## 8.5. Quick pre-interview check

Run these to be confident everything lines up:
```bash
npm run check-assets   # ✅ every runtime asset is in sw.js's ASSETS
npm run test:i18n      # ✅ every {vi,en} has both halves
npm run serve          # then open http://localhost:8000 and click through the Reasoning room
```
