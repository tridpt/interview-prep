# 2. Foundational Concepts (theory)

This file explains **every term** in the project, written for someone starting from scratch. It has two parts: **(A) web/technical concepts** used to build it, and **(B) AI concepts** the rooms illustrate. Master this and you'll be confident against any "concept" question.

---

# PART A — Web & technical concepts

## 2.1. ES Modules (import / export)

**What is it?** Modern JavaScript's standard module system: each `.js` file is a module, sharing code via `export` and `import`. Here `app.js` is loaded as `<script type="module">`, then:

```js
import { renderHome } from "./rooms/home.js";
import { getLang, setLang, tx } from "./i18n.js";
export const ROOMS = [ /* … */ ];
```

**Why an HTTP server (not `file://`)?** The browser loads modules via `fetch`, and `fetch` is blocked for the `file://` protocol for security (CORS). So you must serve over `http://localhost` (e.g. `python -m http.server`).

**Benefit for the project:** each room is an isolated module, no global leakage, dependencies explicit via `import`. No bundler/build needed — the browser loads them.

> 💡 Compared to a "giant single-IIFE file", this is more modern: ES Modules give scope isolation *and* file splitting, without a build step.

---

## 2.2. PWA (Progressive Web App)

**What is a PWA?** A web app "upgraded" to behave almost like a native app: installable to the home screen, its own icon, fullscreen, and **works offline**.

**Three parts of a PWA (all present here):**
1. **HTTPS** (or localhost) — required to register a service worker.
2. **Web App Manifest** (`manifest.json`) — JSON declaring name, icons, theme color, display mode.
3. **Service Worker** (`sw.js`) — a background script that intercepts requests to serve from cache → offline.

**Here:** the PWA makes AI Explorer installable as an AI-learning app and **fully usable offline** — every room is on-device JS/canvas computation, no network calls.

---

## 2.3. Service Worker & Cache API & cache versioning

**What is a Service Worker (SW)?** A JavaScript file running in the "background", separate from the page, acting as a **client-side network proxy**: it intercepts every request and decides to serve from cache or go to the network.

**Lifecycle:**
- **install** — precache the `ASSETS` list into the Cache API.
- **activate** — clean up old caches with a different version.
- **fetch** — every request passes through; the project uses **cache-first** for same-origin assets (serve cache if present, else network then cache), falling back to `index.html` when offline.

**Cache versioning — the most important technique:** the project uses `const CACHE = "ai-explorer-v22";`. Every time a file changes you must **bump** the version (v22 → v23). When the new SW `activate`s, it deletes every cache whose name isn't `CACHE` and loads the new one. This keeps users from getting stuck on an old version.

**A subtle detail here — no auto `skipWaiting`:**
```js
self.addEventListener("install", (e) => {
  // NO auto skipWaiting: let the new worker "wait" until the user clicks "Reload".
  e.waitUntil(caches.open(CACHE).then((c) => c.addAll(ASSETS)));
});
self.addEventListener("message", (e) => {
  if (e.data === "SKIP_WAITING") self.skipWaiting();
});
```
The new SW deliberately **waits** instead of taking over immediately, so assets don't change mid-use. Only when the user clicks "Reload" on the toast (pwa.js posts `SKIP_WAITING`) does the new worker activate and the page reload.

> 💡 **Common question: "Why doesn't the SW run over `file://`?"** For security, a SW only runs over `https://` or `localhost`.

---

## 2.4. Client-side routing via URL hash

**What is it?** Changing the "page" (room) **without a full reload**. The project uses the URL's **hash** (`#/tokenizer`):

```js
window.addEventListener("hashchange", route);
export function navigate(id) {
  location.hash = id === "home" ? "" : `#/${id}`;
}
```
Changing the hash doesn't reload the page, it just fires `hashchange` → the `route()` function reads the current room and renders. The hash is also the natural place to **deep-link** (see 2.5) while working on static hosting like GitHub Pages (no server rewrite config needed).

---

## 2.5. State deep-linking (roomstate.js)

**Problem:** opening a room is easy, but sharing *a specific demo* (the sentence being typed in Tokenizer, the problem chosen in Reasoning) needs to save **state** in the URL.

**Solution:** encode state into the **hash query**, e.g. `#/reasoning?p=2` or `#/tokenizer?text=Hello`. `roomstate.js` provides:
- `parseHash()` → splits the hash into `{ id, params }`.
- `getParam(key, fallback)` → reads one parameter.
- `setParams(obj)` → writes params via `history.replaceState` (does NOT trigger re-navigation, avoiding a re-render).
- `buildShareUrl(id, obj)` → builds an absolute URL to share.

> 💡 Nice touch: using `replaceState` (not `hashchange`) updates the URL "silently" as the user types/selects, so it doesn't re-render the room.

---

## 2.6. Hand-written i18n (internationalization)

**What is i18n?** Designing an app to support multiple languages (abbreviated because there are 18 letters between the "i" and "n" of *internationalization*).

**Mechanism here (`i18n.js`, no library):**
- A `lang` variable read from `localStorage` (defaults to `"vi"`).
- `tx(obj)` — takes a `{ vi, en }` object, returns the string for the current language.
- `tx(a, b)` — the two-argument form, quick pick between two strings.
- `setLang(l)` — switches language, saves to localStorage, fires a `"langchange"` event.

```js
export function tx(a, b) {
  if (a && typeof a === "object") return a[lang] ?? a.vi; // tx({vi,en})
  return lang === "en" ? b : a;                            // tx("vi","en")
}
```

When `langchange` fires, `app.js` rebuilds the toolbar and **re-renders the current room** so every dynamic string switches. All bilingual content lives right in the data (each room's `title/question/blurb` is `{vi,en}`).

> 💡 A dedicated test (`i18n.mjs`) scans every `{vi,en}` object to catch a forgotten half — see file 05.

---

## 2.7. localStorage & state persistence

`store.js` wraps all localStorage reads/writes under one key `"ai-explorer-progress"` (JSON):
- `visited` — list of rooms visited.
- `bestScore` — highest final-quiz score.
- `quizAnswered` — `{ roomId: [indices of solved micro-quizzes] }`.
- `track` — the selected track (`beginner`/`full`/`dev`).

Every read/write is wrapped in `try/catch` so it never breaks in private mode (Safari private mode blocks localStorage).

---

## 2.8. Canvas, accessibility & other terms

- **Canvas API**: draw 2D graphics in code — used for the neural net (decision boundary), diffusion, confetti.
- **`prefers-reduced-motion`**: an OS signal that the user wants less motion; animated rooms (e.g. Reasoning revealing steps) check it and drop the delay when set.
- **Focus trap**: keep Tab/Shift+Tab cycling inside the search dialog and return focus to the opener when closing — standard a11y for modals.
- **ARIA**: attributes that help screen readers understand the UI (`role="dialog"`, `aria-modal`, `aria-current`, `aria-label`…).
- **Computed WCAG contrast**: `app.js` has `luminance()`, `contrast()`, and `darkenForText()` to **progressively darken** the theme color until it hits a contrast ratio ≥ 4.5:1 against the light background — so colored text stays readable (see file 05).
- **App shell**: the minimal HTML/CSS/JS shell that's precached to run offline.
- **Web Audio synthesis**: create sound with oscillator + gain, no media files.

---

# PART B — AI concepts the rooms illustrate

Each room is one concept. Here's the "theory ammo" so you can explain any room if asked "what does this room teach".

| Room | AI concept | One-liner |
|------|------------|-----------|
| Teachable | **KNN / learning from examples** | AI learns from *sample data*, not hand-written rules; classifies by nearest neighbor |
| Neural-net | **Neural network / MLP** | Many neurons combined tune their weights to learn complex shapes (deep learning) |
| Overfitting | **Overfitting** | AI "memorizes" old data, aces train but fails test; needs to generalize |
| Decision-tree | **Decision tree** | A *transparent* AI — a chain of yes/no questions where you see each rule |
| Reinforcement | **Reinforcement learning (RL)** | Learns by reward/penalty through trial and error, no answer key |
| Clustering | **Unsupervised / k-means** | Groups similar data by itself, with no labels |
| Tokenizer | **Tokens & tokenizer** | AI slices text into *tokens*, bills by tokens (not letters) |
| Embeddings | **Embeddings / meaning vectors** | Word → point on a map; distances carry meaning ("king − man + woman = queen") |
| Attention | **Attention mechanism** | Each word "looks at" which others to grasp context (the Transformer foundation) |
| Next-token | **Next-token prediction / hallucination** | AI generates text by guessing the next word probabilistically → sometimes "hallucinates" |
| Diffusion | **Diffusion (image generation)** | Denoises random noise step by step into an image |
| Recommendation | **Recommender systems** | Predicts taste from like/skip behavior (TikTok/YouTube/Netflix) |
| Bias | **Bias** | AI learns biases present in human data |
| Adversarial | **Adversarial attacks** | Invisible noise makes AI misread — a robustness/safety lesson |
| Turing | **Human or AI?** | Sharpen your eye for human vs AI writing |
| Chatbot | **LLM pipeline** | Combine tokens + meaning + attention + guessing into an assistant |
| RAG | **Retrieval-Augmented Generation** | Retrieve relevant passages → put in context → answer with sources, less making-up |
| Finetune | **Prompting vs Fine-tuning** | Guide at use-time vs retrain weights — when to use which |
| Agents | **AI agents / tool-use** | AI plans → calls tools → reads results → loops until done |
| Multimodal | **Multimodal** | Images + text mapped into one "meaning space" |
| Context-window | **Context window** | AI holds only a limited number of tokens; old ones fall out → "forgets" |
| Prompt-injection | **Prompt injection & safety** | A "sneaky instruction" fools AI because system and user text mix into one stream |
| RLHF | **RLHF** | Humans rank answers → a reward model → the AI learns to behave helpfully |
| Energy | **Energy cost** | Each use is small, but times millions it's huge |
| **Reasoning** | **Chain-of-thought / reasoning models** | "Thinking" step by step beats "answering instantly" — see 2.9 |
| Summary | **Recap** | Revisit the core ideas |

## 2.9. Chain-of-thought & reasoning models (the new room — IMPORTANT)

This is room 25 (`reasoning.js`) and the **modern highlight** of the project — know it cold.

**What is chain-of-thought (CoT)?** A technique where a language model **writes out intermediate reasoning steps** before giving the final answer, instead of answering instantly. For "how many *r*'s are in *strawberry*": answering instantly often blurts "2" (wrong), but spelling it out, marking each *r*, then counting gives "3" (right).

**Why does "thinking" make it more accurate?** The core: a model generates text by guessing the next word (see the Next-token room). Forced to answer instantly, it only gets "one guess". When it writes step by step, **each written step becomes context for the next**, so the solution is *built up* rather than guessed in one shot. In other words, the model isn't "smarter" — it just gets **more room to compute**.

**What are reasoning models?** Models (like OpenAI's **o1**, **DeepSeek-R1**) trained to **automatically** generate a long (often hidden) chain of thought before committing to an answer — without being told to "think step by step". CoT is a *prompting technique*; a reasoning model is a *model trained to internalize that technique*.

**Trade-off:** "thinking" costs **more tokens and is slower** (cost + latency). The room illustrates this with a token badge: answering instantly ≈ 1–3 tokens, thinking step by step ≈ 48–70 tokens. So you only want to "think hard" on tough problems (math, logic, multi-step coding); simple ones should answer instantly to save cost/latency.

> 💡 **Interview tip**: if asked "do you keep up with AI trends?", tell them about the Reasoning room: "I added a whole room for *reasoning models* — a visual comparison of 'answer instantly' (often wrong) vs 'think step by step' (more accurate), with a token badge showing the speed/cost ⟷ accuracy trade-off. That's the idea behind o1 and DeepSeek-R1."
