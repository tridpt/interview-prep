# 7. Cheat Sheet — Quick Review Before Go-Time

A one-pager. Skim for 5-10 minutes right before the interview.

---

## 🎯 30-second pitch (memorize)
> AI Explorer: an "interactive AI museum" of **26 rooms** running 100% in the browser, each illustrating an AI concept (neural nets, tokens, embeddings, attention, RAG, agents, RLHF, **chain-of-thought**…) and letting you play hands-on. **Vanilla JS + ES Modules — no backend, no API keys**, a **PWA that runs offline**, **bilingual VI/EN**, Neo-brutalism UI. **Modular** architecture: each room is a file exporting `render(root)`, added to the `ROOMS` array; CI's three test layers **block deploy on failure**.

## 🧱 Tech stack
- **Core**: Vanilla HTML/CSS/JS (ES6+) + **ES Modules**, no build step.
- **PWA**: Web App Manifest + Service Worker + Cache API.
- **Web APIs**: hash routing + `hashchange`, `history.replaceState` (deep-link), localStorage, Web Audio (synth), Canvas, getUserMedia, Web Share/Clipboard, `matchMedia(prefers-reduced-motion)`.
- **Test/CI**: Playwright (smoke), Node scripts (check-assets, i18n), GitHub Actions → GitHub Pages.
- **Dev tools**: Python + Pillow (gen_icons.py, gen_og.py — not runtime).

## 🗂️ Files
| File | Role |
|---|---|
| `app.js` | Orchestrator: `ROOMS`, `hashchange` router, `route()`, THEME, toolbar, search, share |
| `i18n.js` | Bilingual: `tx({vi,en})` / `tx(a,b)`, `setLang` fires `langchange` |
| `store.js` | localStorage `ai-explorer-progress`: visited, bestScore, quizAnswered, track |
| `roomstate.js` | Deep-link: `parseHash`, `getParam`, `setParams` (replaceState), `buildShareUrl` |
| `roomquiz.js` | `BANK` + `renderMicroQuiz`, `hasMicroQuiz` |
| `tracks.js` | `TRACKS` (beginner/full/dev) + `trackRoomIds` |
| `sound.js` | Web Audio synth (`sfx`) + `celebrate()` confetti |
| `pwa.js` | Registers SW + "new version" toast + install button |
| `sw.js` | `CACHE="ai-explorer-v22"` + `ASSETS`, cache-first, no auto skipWaiting |
| `rooms/*.js` | Each exports `render(root)`; cleans up via `roomleave` |

## 📐 Architecture
Client-side SPA, **no backend**. `app.js` is the frame — rooms own the guts. `ROOMS` = single source of truth (story arc). Hash router. Each room computes AI on-device (KNN/MLP/k-means/bigram/simulation) → offline.

## 🌊 Service Worker (know it cold)
- **install** → precache `ASSETS` (NO auto skipWaiting).
- **message** → on `SKIP_WAITING` call `skipWaiting()`.
- **activate** → delete old caches ≠ `CACHE` → `clients.claim()`.
- **fetch** → cache-first same-origin; network for the rest; fallback `index.html`.
- Change assets → **bump `CACHE`** (v22→v23). SW only over `http(s)://`.
- Update: "New version — Reload" toast → `SKIP_WAITING` → `controllerchange` → reload.

## 🧩 ES Modules
`import`/`export`; one module per room. Must serve over HTTP (fetch blocks `file://`). No bundler.

## 🌐 i18n
`lang` (localStorage) + `tx()`; bilingual content lives in the data ({vi,en}); switching language → `langchange` → rebuild toolbar + re-render room. `i18n.mjs` test scans every {vi,en}.

## 🔗 Deep-link
State into the hash query (`#/reasoning?p=2`); `setParams` uses `replaceState` (no re-render); Share button carries the URL (Web Share/clipboard).

## ♿ Accessibility
`prefers-reduced-motion`; ARIA (dialog/listbox/current); **focus trap** on search + returns focus on close; keyboard nav; **computed WCAG contrast**: `luminance`+`contrast`+`darkenForText` → `--accent-ink` ≥ 4.5:1.

## 🧪 Testing & CI (blocks deploy on failure)
1. `check-assets.mjs` — every runtime asset must be in `ASSETS` (prevents silent offline breakage).
2. `i18n.mjs` — every `{vi,en}` has both halves (character walker, not regex).
3. `smoke.mjs` — Playwright walks every room (reads `ROOMS`), fails on console errors.
4. `focus-trap.mjs`, `pwa.mjs`.

## 💭 Reasoning room (highlight — know it cold)
- Room 25, chain-of-thought / reasoning models (o1, DeepSeek-R1).
- **Answer instantly** (wrong, red, ≈1–3 tokens) vs **Think step by step** (right, green, ≈48–70 tokens).
- Core idea: the model isn't "smarter", it just gets **more room to compute** — each step is context for the next. Trade-off: slower + more tokens.
- CoT = prompting technique; reasoning model = model trained to think on its own.
- Deterministic, bilingual, deep-link `p`, has a quiz, in the *For developers* track.

## 🤖 AI concepts by room (one line)
KNN (learn from examples) · MLP (deep learning) · overfitting (memorizing) · decision tree (transparent) · RL (reward/penalty) · k-means (self-grouping) · tokens · embeddings (meaning map) · attention · next-token/hallucination · diffusion · recommendation · bias · adversarial · turing · chatbot · RAG (look up docs) · prompting vs fine-tuning · agents (tool-use) · multimodal · context window · prompt injection · RLHF · energy · **chain-of-thought**.

## ⚖️ Main trade-off
No-framework/no-build → light, great offline, no-deps; at the cost of writing my own router/i18n/PWA + managing room lifecycles (offset with `roomleave` + tests).

## ⚠️ Known limitations (raise proactively)
The "AIs" are small simulations/algorithms to build intuition (not real models); the webcam needs permission + localhost/https; no unit tests yet for each room's algorithm.

## 💬 Key terms
- **PWA / Service Worker / Cache API / cache versioning / app shell**
- **ES Modules / hash routing / history.replaceState / deep-link**
- **i18n / langchange / localStorage**
- **Web Audio / autoplay policy / Canvas / getUserMedia**
- **prefers-reduced-motion / ARIA / focus trap / WCAG contrast**
- **chain-of-thought / reasoning models / token trade-off**
- **RAG / embeddings / attention / RLHF / fine-tuning vs prompting / agents**

---

### ✅ Pre-interview mental checklist
- [ ] Pitch the project in 30 seconds.
- [ ] Explain the service worker lifecycle + the never-stuck update flow.
- [ ] The reason for vanilla JS + ES Modules (offline-first, no-deps).
- [ ] Explain chain-of-thought & the token trade-off.
- [ ] Name the three test layers and why `check-assets` matters.
- [ ] Describe how to add a new room (extensibility).
- [ ] Prepared 2 questions to ask the interviewer.

**Breathe. You understand this project. Tell your story with confidence. 🍀**
