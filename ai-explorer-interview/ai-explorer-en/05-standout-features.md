# 5. Standout Features (the point-scorers)

These are the features worth raising proactively to impress. Each has: **problem → solution → why it's impressive**.

---

## 5.1. Fully offline, no backend, no API keys

- **Problem**: AI-learning apps usually depend on a server/API — lose the network and they break, and they need keys (costs money, leaks secrets).
- **Solution**: every room **computes on-device** in JS/canvas (KNN, MLP, k-means, bigram, diffusion simulation…), and the service worker caches the whole app shell → fully offline.
- **Why impressive**: "Not a single network call, no API key — every 'AI' in the app is a real algorithm running on the user's machine, so it's private, free, and offline."

---

## 5.2. Modular architecture, easy to extend

- **Problem**: 26 rooms could turn `app.js` into spaghetti.
- **Solution**: each room is an isolated `.js` module exporting `render(root)`; `app.js` only orchestrates via the `ROOMS` array (a single source of truth). Adding a room = create a file + one `ROOMS` entry + one `ASSETS` entry.
- **Why impressive**: "The frame (title, progress, nav, quiz) is handled automatically; rooms only own content. That's separation of concerns without a framework."

---

## 5.3. Full VI/EN bilingual + a test guarding it

- **Problem**: a missing translation is a silent bug, hard to spot among thousands of strings.
- **Solution**: `i18n.js` with `tx({vi,en})`; switching language fires `langchange` → instant re-render. The `i18n.mjs` test **scans character by character** over every `{vi,en}` object to catch a missing half or an empty string.
- **Why impressive**: "I didn't rely on naive regex (wrong with nested template literals) — I wrote a character walker with a stack that knows which object each `vi`/`en` key belongs to, then put it in CI."

---

## 5.4. PWA with an update notice (never stuck on old)

- **Problem**: aggressive service-worker caching easily leaves users **stuck on an old version** forever.
- **Solution**: the new SW deliberately does **not** auto `skipWaiting` — it "waits"; `pwa.js` shows a **"New version — Reload"** toast; clicking it posts `SKIP_WAITING`, the new worker takes over, and the page auto-reloads.
- **Why impressive**: "I prioritized a stable experience — no assets changing mid-use — while still letting users opt into the update with one click. Plus cache versioning (`ai-explorer-v22`) cleanly purges old caches."

---

## 5.5. Deep-links that share an exact state

- **Problem**: sharing "the Tokenizer room" works, but sharing *the exact sentence I'm typing* doesn't.
- **Solution**: `roomstate.js` encodes state into the hash query (`#/reasoning?p=2`), using `replaceState` to update silently; the Share button carries that URL (Web Share API/clipboard).
- **Why impressive**: "The URL becomes shareable state, and it still works on static hosting (GitHub Pages) without server rewrites."

---

## 5.6. Quick room search with a focus trap (accessibility)

- **Problem**: 26 rooms need quick lookup; a modal without focus management is a classic a11y bug.
- **Solution**: the `/` key opens search; filters by name + keywords (accent-normalized via `normalize("NFD")`); ↑↓/Enter/Esc; a **focus trap** keeps Tab inside the dialog and **returns focus to the opener** on close; full ARIA (`role="dialog"`, `aria-modal`, `role="listbox/option"`, `aria-activedescendant`).
- **Why impressive**: "I handle focus like a real library component, in plain JS."

---

## 5.7. Computing color contrast to WCAG

- **Problem**: each room has a vivid tone — that colored text on a light background often **fails contrast** (hard to read, fails WCAG).
- **Solution**: `luminance()` + `contrast()` (the exact WCAG formula) + `darkenForText()` progressively darkens the color (×0.9/loop) until the ratio ≥ 4.5:1, assigned to a `--accent-ink` variable used only for text. The color blocks keep their vividness.
- **Why impressive**: "Accessibility isn't an afterthought — I compute contrast in code so every room is readable while keeping the Neo-brutalism aesthetic."

---

## 5.8. CI blocks deploy if tests fail

- **Problem**: a many-file project is easy to "forget" — forget to cache an asset (silent offline breakage), forget a translation, or a new room throws a console error.
- **Solution**: GitHub Actions (`deploy.yml`) runs a `test` job before deploy:
  1. `check-assets.mjs` — every runtime asset must be in `sw.js`'s `ASSETS`.
  2. `i18n.mjs` — every `{vi,en}` has both halves.
  3. `seo.mjs` — checks bilingual prerendered pages/sitemap from `build-seo.mjs`.
  4. `smoke.mjs` — Playwright walks **every room** (reading `ROOMS` from source), failing on console errors/exceptions.
  5. `focus-trap.mjs`, `pwa.mjs`, `a11y.mjs` — focus, PWA and accessibility.
  A separate Lighthouse job checks SEO/accessibility budgets and tracks performance; deploy waits for both jobs.
  A failure in any step → **deploy is blocked**.
- **Why impressive**: "`check-assets` is the safety net for offline — if I add a room but forget to cache the file, CI errors immediately instead of letting offline break silently. And the smoke test reads `ROOMS`, so it never 'forgets to test' a new room."

---

## 5.9. ⭐ The Reasoning room — chain-of-thought (the timely highlight)

- **Problem**: much AI material hasn't caught up with the *reasoning models* wave (o1, DeepSeek-R1).
- **Solution**: room 25 lets the user pick a "tricky" problem (count the *r*'s in *strawberry*, a CRT-style pen–eraser problem, a day-of-week calculation), then compares directly:
  - **⚡ Answer instantly** → a gut answer, usually **wrong**, marked red, with a badge of **≈1–3 tokens**.
  - **💭 Think step by step** → reveals each reasoning step one by one (with a delay, respecting `prefers-reduced-motion`), then the **correct** answer, marked green, badge of **≈48–70 tokens**.
- **Why impressive**: "This teaches the core idea of reasoning models: give the model *room to compute* — each written step becomes context for the next — and the **token badge shows the speed/cost ⟷ accuracy trade-off**. It's deterministic, bilingual, has a deep-link (param `p`), a micro-quiz, and sits in the *For developers* track. It's proof I track the trend and turn it into a learnable experience."

> 💡 **Interview tip**: use the Reasoning room as your "headline story". It shows both engineering (a new module, timers, deep-link, a11y) and up-to-date AI understanding (CoT, reasoning models, token trade-offs).
