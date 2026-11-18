# 6. Interview Question Bank & Model Answers

40+ questions, grouped, easy to hard. Each has a **suggested answer**. Read then re-phrase in your own words.

> ⭐ = frequently asked. Prioritize these.

---

## A. Overview & motivation (warm-up)

**A1. ⭐ Introduce the project briefly.**
> AI Explorer is an "interactive AI museum" of 26 rooms that runs entirely in the browser. Each room explains an AI concept visually and lets you play hands-on — from a webcam teachable machine, neural nets, tokens, embeddings, attention, to RAG, agents, RLHF, and even chain-of-thought. 100% vanilla JS + ES Modules, no backend, no API keys, a PWA that runs offline, bilingual VI/EN.

**A2. Why did you build it?**
> I wanted to help beginners *understand AI intuitively* rather than reading formulas, and to challenge myself to build everything in vanilla JS: router, i18n, PWA, deep-linking, even the test pipeline. Each room is a little game illustrating one concept.

**A3. Who are the users?**
> Two groups: complete beginners (the *Beginner* track, 5 core rooms) and developers wanting the concepts to build AI apps (the *For developers* track: tokens, embeddings, RAG, fine-tuning, agents, reasoning).

**A4. Why "understand AI in 15 minutes"?**
> Because the design prioritizes intuition: no math, no dense jargon; each concept is a click-and-see demo. The *Beginner* track takes ~10 minutes end to end.

---

## B. Architecture & design

**B1. ⭐ Describe the overall architecture.**
> A client-side SPA, no backend. `app.js` combines `ROOM_META` with `ROOM_LOADERS` into the `ROOMS` array (the story arc) and loads a room module on demand; it also owns the `hashchange` router, toolbar, and subsystems (i18n, store, sound, roomstate, roomquiz). `pwa.js`+`sw.js` handle offline. Data lives in localStorage.

**B2. ⭐ Why no backend / API key?**
> Not needed — every room computes on-device in JS (KNN, MLP, k-means, bigram, simulations). No backend means: runs offline, fully private, free (no API cost), simple deploy (static GitHub Pages), no server to crash.

**B3. ⭐ Why vanilla JS + ES Modules, not a framework?**
> The goal was a lightweight, offline-first, no-dependency educational app. Vanilla + ES Modules give a tiny app shell to cache, no supply-chain attack surface, no complex build, and each room is an isolated module. The cost is writing my own router/i18n/PWA — also where I learned the most.

**B4. What are ES Modules, and why serve over HTTP?**
> The standard JS module system (`import`/`export`), one module per file. The browser loads modules via `fetch`, which is blocked for `file://` (CORS), so you must serve over `http://localhost`, e.g. `python -m http.server`.

**B5. What role does the `ROOMS` array play?**
> A single source of truth for navigation: order = story arc. The router, progress bar, Prev/Next buttons, home grid, search, and even the smoke test all rely on it. Adding/removing a room is a one-place edit.

**B6. How does `route()` work?**
> On hash change: fire `roomleave` → clear DOM → `applyTheme` → render home or find the room in `ROOMS`, mark visited, build its frame, `await room.render()` to load the module, reject a stale route, call the returned render function, then insert the quiz and navigation.

**B7. Where is app state stored?**
> localStorage under `ai-explorer-progress`: visited rooms, best quiz score, solved micro-quizzes, track. Language and sound settings are in separate keys. All reads/writes wrapped in try/catch for Safari private mode.

**B8. Hash routing or the History API? Why?**
> Hash (`#/tokenizer`) + the `hashchange` event. Hash works directly on static hosting (GitHub Pages) with no server rewrite config, and it's the natural place to attach deep-link state.

---

## C. PWA & Offline (focus area)

**C1. ⭐ What is a PWA? What does it need?**
> A web app installable to the home screen and running offline like a native app. It needs: HTTPS (or localhost), a Web App Manifest, and a Service Worker.

**C2. ⭐ How does the service worker work here?**
> install: precache the `ASSETS` array into the Cache API. activate: delete old caches ≠ `CACHE` then `clients.claim()`. fetch: cache-first for same-origin assets, network for the rest, fallback `index.html` when offline.

**C3. ⭐ What is cache versioning? Why does it matter?**
> The `CACHE = "ai-explorer-v22"` constant. Every file change bumps the version; when the new SW activates it deletes every differently-named cache → users don't get stuck on the old version. It's the core technique for shipping updates with a SW.

**C4. ⭐ How do users get updates without getting stuck?**
> The new SW deliberately does NOT auto `skipWaiting` — it "waits" so assets don't change mid-use. `pwa.js` shows a "New version — Reload" toast; clicking it posts `SKIP_WAITING`; the new worker calls `skipWaiting()` then takes over (`controllerchange`) → the page auto-reloads once.

**C5. Why doesn't the SW run over `file://`?**
> Security — a SW only runs over `https://` or `localhost`. Opening a file directly can't register the SW.

**C6. What's your caching strategy and why?**
> Cache-first for same-origin assets (fast, great offline since assets are essentially static between versions), network for external resources (e.g. Google Fonts), and an `index.html` fallback offline. Since each release bumps the version, cache-first doesn't cause staleness.

**C7. How important is offline for this project?**
> Very — a learning app should work on a plane, weak network, or a no-network demo. Since every room computes on-device, offline is nearly "free" once the app shell is cached.

---

## D. i18n & bilingual

**D1. ⭐ Your multilingual mechanism?**
> A `lang` variable from localStorage; `tx({vi,en})` returns the string for the language, or `tx(a,b)` picks quickly. Bilingual content lives right in the data (each room's title/question/blurb is {vi,en}). Switching language fires `langchange` → rebuild toolbar + re-render room.

**D2. Why hand-write i18n instead of a library?**
> Simple need (2 languages), only ~20 lines to write, avoids a dependency (matching the no-deps philosophy). A full i18n library would be overkill and bloat the app shell.

**D3. How do you ensure no string goes untranslated?**
> The `i18n.mjs` test scans every `.js` file, reconstructs object literals to find every `{vi,en}` and errors if a half is missing or a string is empty. It walks character by character (skipping strings/comments) instead of using regex because strings live in nested template literals.

**D4. Does the open room update when you switch language mid-way?**
> Yes — `setLang` fires `langchange`; `app.js` catches it, rebuilds the toolbar, and calls `route()` to re-render the current room, so every dynamic string switches immediately.

---

## E. JavaScript / vanilla & resource management

**E1. ⭐ Without a framework, how do you manage room lifecycles?**
> `route()` fires a `roomleave` event before switching. Each room listens for `roomleave` (once) to clean up timers/animation/webcam. Not cleaning up leaks — e.g. a metronome/interval running forever or the camera staying on.

**E2. Give an example of a resource leak you handled.**
> The webcam room: on leave you must `stream.getTracks().forEach(t => t.stop())` to turn off the camera; animated rooms must `clearInterval`/`clearTimeout`. Reasoning keeps a `timers` array and clears it on `roomleave` or when a new problem is chosen.

**E3. How do you guard against XSS / unwanted content?**
> Content is dev-controlled (no arbitrary user HTML). Deep-links read a param then coerce it (e.g. `parseInt(getParam("p"))` with a validity check), so no HTML can be injected. If I extended it to accept user input, I'd use `textContent` over `innerHTML`.

**E4. Why `history.replaceState` for deep-links?**
> To update the URL "silently" as the user types/selects WITHOUT firing `hashchange` (avoiding a re-render and avoiding flooding the back history). `replaceState` swaps the current URL in place.

**E5. How is sound produced? How do you handle the autoplay policy?**
> Synthesized with Web Audio: oscillator + gain, no media files. Browsers block audio before interaction, so the `AudioContext` is only created/`resume()`d when needed (after a user click).

---

## F. AI concepts (the rooms)

**F1. ⭐ What does the "Teach an AI yourself" room illustrate?**
> Learning from examples (KNN): the AI learns from the samples you capture via webcam, not hand-written rules. It reduces images to a 24×24 gray grid then classifies by nearest neighbor. Lesson: data is the "fuel", and skewed data means a skewed AI.

**F2. How does the room 02 neural net actually learn?**
> It's a 2→H→1 MLP with hand-written `forward` + manual backprop in `trainStep`. Drag the neuron count: 1 neuron only draws a line so it can't separate a circle; more neurons combine many lines into a curve — illustrating "deep learning".

**F3. Explain overfitting in plain terms.**
> When an AI "memorizes" — it nails training data but fails new (test) data. Fixes: more data, a simpler model, or regularization.

**F4. What is a token? Why bill by tokens?**
> AI doesn't read letters like us — it slices text into tokens (subword chunks). Models process and bill by token count, not letters, so long sentences/rare languages cost more tokens.

**F5. What do embeddings and "king − man + woman = queen" mean?**
> Embeddings turn words into vectors (points on a meaning map); similar words are close, and *distance/direction* carries relational meaning — so vector arithmetic yields analogies.

**F6. What is attention?**
> A mechanism letting each word "look at" other words to grasp context (e.g. what "it" refers to). It's the foundation of Transformers and modern LLMs.

**F7. Why does AI "hallucinate"?**
> Because it generates text by guessing the next word probabilistically, without fact-checking. It picks *plausible-sounding* words, so it sometimes makes things up while sounding confident.

**F8. What is RAG, and what does it solve?**
> Retrieval-Augmented Generation: before answering, the AI *retrieves* relevant passages and puts them in context, so it grounds its answer in your own docs, hallucinates less, and cites sources — without retraining the model.

**F9. How do prompting and fine-tuning differ?**
> Prompting: instruct the model at use-time (no weight change) — cheap, fast, flexible. Fine-tuning: retrain on your data to change weights — costlier but "bakes in" style/knowledge. Prompt/RAG first, fine-tune only when truly needed.

**F10. How does an AI agent differ from a plain chatbot?**
> An agent doesn't just answer — it plans, calls tools (calculator, search…), reads results, and loops over multiple steps until the task is done.

**F11. What is the context window?**
> The max number of tokens a model "remembers" at once. Too long a conversation and old tokens fall out of the window → the AI "forgets" the beginning.

**F12. What is prompt injection, and why does it fool AI?**
> Slipping a "sneaky instruction" into content to make the AI break its original instructions. It works because the AI reads system and user text as one stream, making it hard to tell which is the "real command".

**F13. What is RLHF?**
> Reinforcement Learning from Human Feedback: humans rank answers → train a "reward model" on those preferences → tune the model to answer more helpfully and politely. It's the "teach the AI to behave" step.

---

## G. Chain-of-thought & reasoning models (new room — focus)

**G1. ⭐ What is chain-of-thought (CoT)?**
> Having the model *write out intermediate reasoning steps* before the final answer, instead of answering instantly. E.g. counting "r"s in "strawberry": answering instantly tends to give "2" (wrong); spelling → marking → counting gives "3" (right).

**G2. ⭐ Why does making the model "think" step by step improve accuracy?**
> Because the model generates text by guessing the next word. Answering instantly is just "one guess". When it writes step by step, each step becomes context for the next, so the solution is *built up* instead of guessed once. The model isn't smarter — it gets more *room to compute*.

**G3. ⭐ How does CoT differ from reasoning models (o1, DeepSeek-R1)?**
> CoT is a *prompting technique* ("think step by step"). A reasoning model is a *trained model* that automatically generates a long (often hidden) chain of thought before committing — no prompting needed. In other words, a reasoning model internalizes CoT.

**G4. ⭐ What's the trade-off of making the model "think"?**
> Slower and more tokens (cost + latency). The room shows it with a token badge: answering instantly ≈1–3 tokens, thinking step by step ≈48–70 tokens. So you only "think hard" on tough problems.

**G5. When should you use reasoning / CoT, and when not?**
> Use it for: math, logic, multi-step coding, multi-hop reasoning. Skip it for: simple questions, quick lookups, latency/cost-sensitive tasks — because "thinking" there is wasteful, slow, and token-heavy.

**G6. How did you implement the Reasoning room (technically)?**
> Deterministic: each problem has a preset gut answer (wrong) + reasoning steps + correct answer + illustrative token counts. "Answer instantly" shows a red answer immediately; "Think step by step" reveals steps via `setTimeout` (respecting `prefers-reduced-motion` — instant if set), then the green answer. The problem is deep-linked via param `p`, plus a micro-quiz. Timers are cleared on leave/problem change.

---

## H. Testing & CI/CD

**H1. ⭐ How is the project tested?**
> Three layers, all in CI before deploy: (1) `check-assets.mjs` ensures every runtime asset is in `sw.js`'s `ASSETS`; (2) `i18n.mjs` ensures every `{vi,en}` has both halves; (3) `smoke.mjs` uses Playwright to walk every room and fails on any console error/exception. Plus `focus-trap.mjs` and `pwa.mjs`.

**H2. ⭐ What problem does `check-assets` solve?**
> Silent offline breakage: if you add a file but forget to add it to `ASSETS`, offline will be missing that file with no dev-time error. This test reads `ASSETS` from `sw.js`, compares against every real `.js`/core/icon, and fails if any are missing — blocking deploy.

**H3. Why does the smoke test read `ROOMS` from source?**
> So new rooms are *automatically* tested without editing the test. It extracts `id`s from the `ROOMS` array, walks each room in Chromium, and catches any console error/exception/failed request.

**H4. How is it deployed?**
> GitHub Actions: the `test` job runs first, and only on pass does the `deploy` job push to GitHub Pages. After changing assets, remember to bump `CACHE` so clients get the new version.

**H5. Why write a character walker for `i18n.mjs` instead of regex?**
> Because bilingual strings live in multiline template literals, arrays, and with nested quotes — regex would give wrong results. The character walker has a stack that knows exactly which object each `vi`/`en` key belongs to.

---

## I. Accessibility

**I1. ⭐ What do you do for accessibility?**
> Respect `prefers-reduced-motion` (drop animation delays, e.g. in Reasoning); ARIA for the toolbar/progress/search; a focus trap for the search dialog; keyboard navigation (←/→, `/`, ↑↓/Enter/Esc); and computed WCAG contrast for text.

**I2. Explain how you compute color contrast.**
> `luminance()` computes relative luminance per WCAG; `contrast()` computes the ratio; `darkenForText()` progressively darkens the theme color until the ratio against the light background is ≥ 4.5:1, assigned to `--accent-ink` used only for text. The color blocks keep their vividness.

**I3. What is a focus trap and how did you build it?**
> Keeping Tab/Shift+Tab cycling inside the dialog (the search input is the only focusable element), and returning focus to the button that opened the dialog on close (save `document.activeElement` first, `.focus()` it back on close). Plus `role="dialog"`, `aria-modal`.

---

## J. Extensibility & thinking

**J1. ⭐ What does adding a new room take?**
> Create `rooms/x.js` exporting `render(root)`; add an import + `ROOMS` entry; add `THEME`, (optional) `HINTS`, `SEARCH_KEYWORDS`, a `BANK` quiz; add the file to `sw.js`'s `ASSETS` and bump `CACHE`; (optional) add it to a track. The frame is automatic; the smoke test checks it automatically.

**J2. ⭐ Your hardest technical challenge?**
> (Pick one to tell deeply) computing WCAG contrast per room color; or the focus trap + ARIA for search; or the "never stuck on old" PWA update flow; or the character walker for the i18n test. Tell it problem → solution → result.

**J3. The project's biggest trade-off?**
> No-framework/no-build gives lightness, great offline, no-deps — at the cost of writing my own router/i18n/PWA and managing room lifecycles myself (leak-prone if you forget cleanup). I offset it with the `roomleave` convention + automated tests.

**J4. What would you improve with more time?**
> Unit tests for each room's algorithm; more languages (the i18n structure is ready); split CSS per room; and maybe add trend-driven rooms (e.g. MoE, advanced tool-calling).

**J5. The room you're proudest of, and why?**
> The Reasoning room — it's both current (reasoning models o1/DeepSeek-R1) and packs a lot of technique: instant vs step-by-step comparison, token badges showing the trade-off, deep-linking, a11y, a quiz. It proves the modular architecture really is easy to extend.

**J6. Where does the project "break" in feature-limited environments?**
> No webcam permission → room 01 shows a notice and the explanation still reads; no Web Share → falls back to copying the link; no Web Audio → sound is silent (no error). I designed graceful fallbacks nearly everywhere.

---

## 🎤 Top 5 most-asked — practice first
1. A1 — Introduce the project.
2. B3 — Why vanilla JS + ES Modules, no framework.
3. C2/C4 — Service worker & the never-stuck update flow.
4. G1/G2 — What chain-of-thought is, why "thinking" is more accurate.
5. H1/H2 — The three test layers & why `check-assets` matters.

## ❓ Questions to ask the interviewer back
- Does the team do PWA/offline-first, and what caching strategy?
- What's the team's test & deploy-gating (CI) process?
- How much does the team prioritize accessibility in the product?
