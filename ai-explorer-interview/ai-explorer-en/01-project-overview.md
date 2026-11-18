# 1. Project Overview — AI Explorer

## 1.1. What is AI Explorer?

AI Explorer (tagline: *Understand AI in 15 minutes*) is an **"interactive AI museum"** that runs entirely in the browser. Users walk through **26 "rooms"**, each explaining one core AI concept visually and letting you **play hands-on** — not just read theory.

Picture this: you turn on your webcam and teach an AI to tell poses apart in 30 seconds (room 01), drag a slider to watch a neural network reshape its decision boundary (room 02), type a sentence to see how the AI slices it into tokens (room 07), then move on to "advanced" developer rooms like RAG, fine-tuning, agents, and the newest one — the **"AI that thinks step by step" room** (chain-of-thought / reasoning models like o1, DeepSeek-R1).

### Core points to remember
- It's a **PWA (Progressive Web App)**: installable to the home screen like a real app, and **runs offline**.
- **No backend, no API keys, no framework, no build step** — 100% plain HTML/CSS/JS with **ES Modules**.
- **Modular architecture**: each room is a standalone `.js` module exporting a `render(root)` function; `app.js` only orchestrates.
- **Fully bilingual VI/EN** (UI + each room's content), instant language switch.
- **Neo-brutalism** UI (thick black borders, hard shadows, bright color blocks; Archivo + Space Mono fonts).
- Data (progress, score, track) stored locally in **localStorage**, never sent anywhere.

## 1.2. What problem does it solve?

1. **AI is seen as an opaque, jargon-heavy "black box"** → turn each concept into an interactive demo you can "see and touch", no math required.
2. **Newcomers get overwhelmed by 20+ topics** → **learning tracks by level** (Beginner / Full / For developers) guide you to exactly what you need.
3. **AI learning material is often static and dry** → each room is interactive, with a micro-quiz to check understanding, sound feedback, and a story arc throughout.
4. **Demonstrating technical skill**: mastering ES Modules, offline-first PWA, hand-written i18n, accessibility (including computing color contrast to WCAG), deep-linking, and a CI/CD pipeline with tests that block deploy — all in vanilla JS with no dependencies.

> 💡 **Interview tip**: when asked "what is this for", emphasize the **intersection of educational value and technical difficulty**: "I wanted something that helps beginners *understand AI intuitively* — each concept is a little game — while challenging myself to build everything in vanilla JS: router, i18n, PWA, deep-linking, and an automated test suite."

## 1.3. Main features

| Group | Features |
|-------|----------|
| **26 interactive rooms** | Each an AI concept + a theme color; hands-on (webcam, sliders, typing, choosing answers…) |
| **Learning tracks** | *Beginner* (5 core rooms), *Full*, *For developers* — no overwhelm |
| **Per-room micro-quiz** | "Quick check" at the end of a room, adds to the journey score |
| **Final quiz** | Last room + tiered badge, exportable as a PNG image |
| **Quick room search** | Press `/` or 🔍 to find by name/concept, with a **focus trap** for the dialog (a11y) |
| **State deep-links** | Some rooms encode state in the URL (sentence in Tokenizer/Attention, problem in Reasoning…) to share an exact demo |
| **Share button** | Web Share API on phones, copies the link (with state) on desktop |
| **Bilingual VI/EN** | Whole UI + content, instant switch (no page reload) |
| **PWA & offline** | Installable, runs offline, shows a "New version — Reload" toast |
| **Sound & confetti** | Synthesized with Web Audio (no media files), confetti on completion |
| **Accessibility** | Respects `prefers-reduced-motion`, ARIA, computes WCAG contrast for text |
| **Presentation** | Fullscreen mode, ←/→ arrow-key navigation |

## 1.4. Tech stack

**Core:**
- **HTML5 + CSS3 + plain JavaScript (vanilla, ES6+)** with **ES Modules** (`import`/`export`) — no React/Vue, no jQuery, no runtime libraries.
- **No build step**: serve over HTTP and it runs (ES modules must be served over `http(s)://`, not `file://`).

**Browser Web APIs / techniques used:**
- **Service Worker + Cache API** — caching for offline (PWA).
- **Web App Manifest** (`manifest.json`) — declares the PWA (install to home screen, icons, theme color).
- **URL hash + `hashchange`** — client-side router, no page reload.
- **`history.replaceState`** — deep-links room state into the hash query.
- **localStorage** — stores progress, quiz score, solved checks, track, language.
- **Web Audio API** — synthesizes feedback sound (no media files).
- **Canvas API** — draws the neural net, diffusion, confetti, share card.
- **getUserMedia (webcam)** — the "Teach an AI yourself" room.
- **`matchMedia('(prefers-reduced-motion)')`** — accessibility.
- **Web Share API + Clipboard API** — sharing a room.

**Testing & CI:**
- **Playwright** (Chromium) for the smoke test.
- Two pure-Node checks (no browser needed): `check-assets.mjs` (cache integrity), `i18n.mjs` (bilingual integrity).
- **GitHub Actions** runs all tests before each deploy; a failing test **blocks deploy**.

**Dev-only tooling (not runtime):**
- **Python + Pillow** — `gen_icons.py`, `gen_og.py` generate the icon/Open Graph images (run once).

**Deployment:** GitHub Pages (static web, deploy = push files via a workflow).

> 💡 **A trap question: "Why no framework?"** Good answer: "The goal was a **lightweight, offline-first, long-lived educational app** with zero library dependencies — each AI concept just needs a canvas and a little plain JS. Vanilla + ES Modules give me a tiny app shell to cache offline, **no supply-chain attack surface**, no complex build, and each room is an isolated module easy to add/remove. The cost is writing my own router, i18n, and PWA update flow — which is also where I learned the most."

## 1.5. How to run (demo flow)

The project uses ES modules, so it **must be served over a local HTTP server** (don't open `file://` directly).

```bash
# Option 1 — Python (available on most machines):
cd d:\AI_App\ai-explorer
python -m http.server 8000
# Open http://localhost:8000

# Option 2 — VS Code: install "Live Server", right-click index.html → Open with Live Server.
```
> Note: room 01 (webcam) needs camera permission and must run over `localhost` or HTTPS — a browser security requirement.

Run the tests:
```bash
npm install
npx playwright install chromium
npm run check-assets   # offline cache integrity (fast, no browser)
npm run test:i18n      # every { vi, en } string has both halves
npm run serve          # window 1: server at http://localhost:8000
npm run test:all       # window 2: assets, i18n, SEO, smoke, focus, PWA, a11y
```

## 1.6. Folder structure

```
ai-explorer/
├─ index.html          # Page shell + share meta + favicon
├─ style.css           # Entire UI (Neo-brutalism)
├─ app.js              # Router + navigation + ROOMS array + toolbar + search + share + i18n glue
├─ i18n.js             # Bilingual VI/EN (tx, getLang, setLang)
├─ store.js            # localStorage: progress, quiz score, track
├─ roomstate.js        # Deep-link: read/write room state in the hash query
├─ roomquiz.js         # Micro-quiz bank + inserts the "quick check" per room
├─ tracks.js           # Learning tracks (Beginner / Full / For developers)
├─ sound.js            # Web Audio sound + confetti
├─ analytics.js        # Anonymous stats (GoatCounter, optional)
├─ pwa.js              # Registers the service worker + install button + update toast
├─ sw.js               # Service worker: offline cache (ASSETS + CACHE version)
├─ manifest.json       # PWA declaration
├─ rooms/              # Each room is a standalone module exporting render(root)
│   ├─ home.js         # Home page + track picker + progress
│   ├─ teachable.js    # Webcam KNN
│   ├─ neural-net.js   # MLP 2→H→1 trained in the browser
│   ├─ … (23 concept rooms) …
│   ├─ reasoning.js    # Chain-of-thought: answer instantly vs think step by step (NEW ROOM)
│   └─ summary.js      # Recap + quiz + badge
├─ data/embeddings.js  # Precomputed 2D vectors for words
└─ tests/              # assets, i18n, SEO, smoke, focus, PWA and a11y checks
```

> 💡 **A point worth raising proactively:** "`rooms-meta.js` serves both the app and SEO prerendering; `room-loaders.js` imports interactive room code only on demand. Adding a room needs metadata, a loader, a module and an offline asset. `npm run test:all` covers SEO, i18n, smoke, accessibility and PWA checks."
