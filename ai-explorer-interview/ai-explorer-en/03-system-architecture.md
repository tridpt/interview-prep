# 3. System Architecture

This file describes **how the pieces fit together**: the startup flow, the router, the `ROOMS` array, the tracks system, and how to add a new room. Master this and you can "tell the project's technical story".

---

## 3.1. Big picture

AI Explorer is a **client-side Single-Page App with no backend**. Everything runs in the browser:

```
index.html  (loads app.js as <script type="module">)
   │
   ├─ app.js  ── central orchestrator
   │     ├─ combines metadata (rooms-meta.js) with lazy loaders (room-loaders.js)
   │     ├─ defines the ROOMS array (the story arc)
   │     ├─ THEME / HINTS / SEARCH_KEYWORDS
   │     ├─ router: hashchange → route()
   │     ├─ toolbar (language / sound / search / present)
   │     └─ glue: i18n, store, sound, analytics, roomstate, roomquiz
   │
   ├─ i18n.js / store.js / roomstate.js / roomquiz.js / tracks.js / sound.js  (subsystems)
   ├─ pwa.js → registers sw.js (offline service worker)
   └─ rooms/*.js  ── each exports render(root), owns its content + interactivity
```

Design principle: **`app.js` owns the frame, each room owns the guts.** `app.js` builds the title, progress bar, nav buttons, and inserts the micro-quiz; the room's interactive content lives in its own module.

`rooms-meta.js` holds lightweight metadata shared by `app.js` and `build-seo.mjs`; `room-loaders.js` dynamically imports room code on demand. `build-seo.mjs` generates static `vi/`, `en/` pages and a sitemap. CI runs `test:all`, prerendering and Lighthouse before deployment; `privacy.html` explains optional analytics.

---

## 3.2. Startup flow

At the end of `app.js`:
```js
buildToolbar();   // build the toolbar (language/sound/search/present)
initAnalytics();  // init anonymous stats (optional)
route();          // render the room for the current hash
```
In parallel, `index.html` loads `pwa.js` → registers the service worker to enable offline and watch for updates.

---

## 3.3. The `ROOMS` array — the heart of navigation

`ROOMS` is an array of objects, whose **order = the journey's story arc**. Each element:

```js
{
  id: "reasoning", icon: "💭", num: "25",
  title:   { vi: "AI biết suy nghĩ từng bước", en: "AI that thinks step by step" },
  question:{ vi: "…", en: "Why does 'thinking' make AI more accurate?" },
  blurb:   { vi: "…", en: "…" },
  render:  ROOM_LOADERS["reasoning"], // async loader returning render(root)
}
```

`ROOMS` is used everywhere:
- The **router** finds a room by `id`.
- The **progress bar** (`renderProgress`) creates a dot per room.
- The **Prev/Next buttons** take `ROOMS[idx-1]` / `ROOMS[idx+1]`.
- The **home page** (`home.js`) renders the room-card grid.
- The **search box** filters over `ROOMS` + `SEARCH_KEYWORDS`.
- The **smoke test** reads `ROOMS` directly from the source to walk every room.

> 💡 Ordering and descriptions live in `rooms-meta.js`; `room-loaders.js` maps IDs to dynamic imports. SEO prerendering reuses the metadata. Adding a room also requires a loader and offline asset entry.

---

## 3.4. `route()` — how a room is built

When the hash changes, `route()` runs in order:

1. Fire a `"roomleave"` event so the old room cleans up (clear timers, stop webcam, stop animation).
2. Clear old content, scroll to top, replay the enter animation.
3. `applyTheme(id)` — set the CSS theme-color variables for the room (see 3.5).
4. `trackView(...)` — record a view (anonymous analytics).
5. If it's `home` → `renderHome(...)` then stop.
6. Otherwise, find the room in `ROOMS`; if missing → back to home.
7. Mark it visited (`markVisited`) + update the progress bar.
8. Build the **shared frame**: room header (icon, question, title) + Share button.
9. `await room.render()` loads the module; check the route has not changed, then call the returned function with `body`. Loading failures show an error.
10. If the room has a micro-quiz → `renderMicroQuiz(body, id)`.
11. Show the onboarding hint (`showHint`).
12. Build Prev / Next nav buttons.

```js
const room = ROOMS[idx];
visited.add(room.id);
markVisited(room.id);
renderProgress(room.id);
// … build head …
const renderRoom = await room.render();
if (generation !== routeGeneration || currentRoute() !== room.id) return;
renderRoom(body);
if (room.id !== "summary" && hasMicroQuiz(room.id)) renderMicroQuiz(body, room.id);
showHint(room.id);
```

---

## 3.5. Theme colors & WCAG contrast (`applyTheme`)

Each room has a color set in `THEME[id] = [accent, accent2, rgb]`. `applyTheme` sets CSS variables:

```js
function applyTheme(id) {
  const [a, a2, rgb] = THEME[id] || THEME.home;
  const r = document.documentElement.style;
  r.setProperty("--accent", a);
  r.setProperty("--accent-2", a2);
  r.setProperty("--accent-rgb", rgb);
  r.setProperty("--accent-ink", darkenForText(a)); // a DARK color used only for text
}
```

The notable bit: `darkenForText()` **progressively darkens** the theme color (×0.9 per loop) until it reaches contrast ≥ 4.5:1 against the light background (using the WCAG luminance/contrast formula). So text in each room's tone stays readable while the color blocks stay vivid — an accessibility detail that scores points.

---

## 3.6. The tracks system (`tracks.js`)

So beginners aren't overwhelmed by 26 rooms, there are 3 tracks:

| Track | Content | Rooms |
|-------|---------|-------|
| `beginner` 🌱 | 5 core rooms, ~10 min | teachable, neural-net, tokenizer, next-token, summary |
| `full` 🧭 | The whole journey | all (rooms = `null`) |
| `dev` ⌨️ | For developers | tokenizer, embeddings, attention, next-token, rag, finetune, agents, multimodal, **reasoning**, chatbot, summary |

`trackRoomIds(trackKey, allRooms)` returns the room ids for a track (fallback: all). The home page (`home.js`) uses it to **dim** off-track rooms and mark "In your path" for on-track ones. The "Start" button jumps to the first unvisited room *within the track*.

> 💡 Note: the **reasoning** room is in the *For developers* track — a way of flagging it as an advanced, current topic.

---

## 3.7. Deep-linking & sharing (with roomstate.js)

- A room writes state via `setParams({ p: idx })` → URL becomes `#/reasoning?p=2` (uses `replaceState`, no re-render).
- When opened, the room reads `getParam("p")` to restore the exact state.
- The "Share" button calls `shareRoom(room)`: uses `navigator.share` on phones, falls back to copying the link (with current state) to the clipboard on desktop.

---

## 3.8. How to add a new room (full checklist)

A classic interview question about extensibility. The steps:

1. **Create `rooms/room-name.js`** exporting a `render(root)` function (by convention named `roomRoomName`). If it has animation/timers, remember to listen for `roomleave` to clean up.
2. **Add an import + one entry to the `ROOMS` array** in `app.js` (in the right story-arc position; fill in `id/icon/num/title/question/blurb` bilingual and `render`).
3. **Add a theme color** to `THEME[id]`.
4. **Add an onboarding hint** to `HINTS[id]` (optional).
5. **Add search keywords** to `SEARCH_KEYWORDS[id]`.
6. **Add a micro-quiz** to `BANK[id]` in `roomquiz.js` (optional).
7. **Add the file path to the `ASSETS` array** in `sw.js` **and bump `CACHE`** (e.g. v22 → v23) — if you forget, `check-assets.mjs` errors out and blocks deploy.
8. (Optional) add it to a track in `tracks.js`.

The frame (title, progress bar, nav buttons, micro-quiz) is handled automatically by `app.js`, and the **smoke test reads `ROOMS`** so the new room is tested right away.

> 💡 The **reasoning** room is a living example of this process — it's the most recently added, and you can point out its fingerprint everywhere: `ROOMS`, `THEME.reasoning`, `HINTS.reasoning`, `SEARCH_KEYWORDS.reasoning`, `BANK.reasoning`, `ASSETS` (`./rooms/reasoning.js`), and `TRACKS.dev`.
