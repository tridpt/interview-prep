# 9. Appendix: Frontend HTML & CSS (read directly from the real files)

> ⚠️ This file was written after **directly reading `frontend/index.html` and `frontend/style.css`** — the two files not analyzed in earlier sections. It completes the agent-talk pack so it covers **100% of every file**.

---

## 9.1. `index.html` — UI structure

`index.html` is compact (~110 lines), with 3 parts: header, main, and a `<template>`.

### Overall layout
```
<header>  → title + description
<main>
  <section class="setup">     → config area (agents, topic, options, buttons)
  <section class="chat">      → conversation display (empty, app.js fills it)
  <div class="interject">     → the "director" interjection box
<template id="agentTpl">      → an agent-card mold (cloned when adding an agent)
<script src="/personas.js">
<script src="/app.js">
```

### The `setup` area — the ids `app.js` operates on
A reference of every element with an `id` (since `app.js` uses `$ = id => document.getElementById(id)`):

| id / class | Type | Role |
|-----------|------|------|
| `#agents` | div | Holds the agent cards (app.js `appendChild`s here) |
| `#addAgentBtn` | button | Add an agent |
| `#topic` | input | The topic (default "Can AI truly have emotions?") |
| `#orderMode` | select | `round` / `auto` (AI picks) / `debate` |
| `#maxTurns` | number input | Turns per run (default 12) |
| `#delay` | number input | Speed in seconds/turn (default 1) |
| `#temp` | number input | Creativity temperature (default 0.9) |
| `#ttsOn` `#memOn` `#searchOn` | checkboxes | Read-aloud / auto-update memory / allow web search |
| `#startBtn` `#continueBtn` `#stopBtn` `#clearBtn` | buttons | Loop controls |
| `.save[data-fmt]` | buttons | Save file .txt/.md/.json (reads `data-fmt`) |
| `#copyBtn` `#shareBtn` `#scoreBtn` | buttons | Copy / share link / score |
| `#sessionInfo` | span | Shows saved-session status |
| `#forgetBtn` | button | Clear the localStorage session |
| `#scoreResult` `#leaderboard` | divs | Where scoring results & the leaderboard are drawn (default `hidden`) |
| `#chat` | section | The chat frame |
| `#interjectInput` `#interjectBtn` | input/button | Director interjection |

➡️ **Tie-in with file 05 (frontend):** every `id` here is what `app.js` references. When `refreshUi()` toggles `disabled`, it iterates exactly these ids.

### `<template id="agentTpl">` — the agent-card mold
`app.js` `addAgentCard()` clones this template. Inside are the classes **app.js reads/writes**:
- `.a-name`, `.a-provider` (LLM brain select), `.a-tpl` (persona-template select), `.a-persona` (personality textarea), `.a-goal` (hidden goal).
- `.mem-box` is a `<details>` (collapsible) containing `.a-memory` (memory textarea) + `.mem-clear` (clear-memory button).
- `.dot` is the color dot (changes via `--c` to distinguish agents).

> 💡 **Why a `<template>`?** A `<template>` doesn't render until cloned by JS (`content.firstElementChild.cloneNode(true)`). It's the way to create repeated elements **without a framework**, cleaner than building HTML from strings.

### Interview notes
- Scripts load in order: `personas.js` (defines `PERSONA_TEMPLATES`) **before** `app.js` (which uses it).
- The HTML **contains no logic** — all behavior is in `app.js`. HTML is just a declarative skeleton.

---

## 9.2. `style.css` — Styling

`style.css` (~270 lines) is plain CSS using **CSS custom properties (variables)** and flex/grid layout. No CSS framework.

### Theme variables & base
```css
:root { --bg:#0f1220; --panel:#1a1f35; --text:#e8eaf2; --muted:#9aa3c0; }
* { box-sizing: border-box; margin:0; padding:0; }
body { max-width: 920px; margin: 0 auto; ... }   /* centered, width-capped */
```
A **dark** theme. CSS variables centralize color changes.

### Main blocks
- **`.agents`**: `display: grid` with `repeat(auto-fill, minmax(240px, 1fr))` → agent cards **auto-arrange into columns** by viewport width (responsive without a media query for this part).
- **`.agent-card`**: has a local variable `--c: #3b82f6` (agent color); `border-top` and `.dot` use `var(--c)`. `app.js` sets `node.style.setProperty("--c", COLORS[i])` for a per-agent color.
- **`.bubble`**: a chat bubble, with `animation: pop .25s` on appear.
  - `.bubble.agent`: left-aligned, left border in the agent's color.
  - `.bubble.director`: centered, brown background, italic (the director's words).
  - `.bubble.error`: red background (error messages).
  - `.bubble.phase`: a debate-phase banner (uppercase, muted).

### The "typing" (streaming) effect — a nice detail
```css
.bubble.speaking .txt::after {
  content: "▋"; animation: blink 1s steps(2, start) infinite;
}
@keyframes blink { to { visibility: hidden; } }
```
➡️ When a bubble has the `speaking` class (during streaming), a blinking cursor `▋` appears at the end — **simulating typing**. `app.js` adds/removes the `speaking` class around the `streamTurn` call.

### Scoring & leaderboard blocks
- `.score-bar span` / `.crit-bar i`: score bars whose `width` is set by `app.js` as a % (smooth `.4s` transition).
- `.score-item.winner`: a gold border (`box-shadow ... inset`) for the winner.
- `.crit-grid`: a 2-column grid of the 4 criteria (logic/evidence/creativity/attitude); collapses to 1 column on mobile.
- `.lb-table`: the leaderboard table, top row (1st place) in larger text.

### Responsive
```css
@media (max-width: 640px) {
  .agents { grid-template-columns: 1fr; }  /* 1 column on mobile */
  .bubble { max-width: 92%; }
  .crit-grid { grid-template-columns: 1fr; }
}
```

### Source footnotes
`.bubble .sources`: a dashed top border, small text; `a` links in light blue `#7dd3fc`. `app.js` `renderSources()` fills links here.

---

## 9.3. Summary: linking HTML/CSS with JS

| Job | HTML/CSS handles | app.js handles |
|-----|------------------|----------------|
| Skeleton & layout | `index.html` + grid/flex | — |
| Per-agent color | the `--c` variable in CSS | sets `--c` by index |
| Chat bubbles | `.bubble.*` classes + animation | creates elements, adds classes, fills text |
| Typing effect | `.speaking` + `@keyframes blink` | adds/removes the `speaking` class during streaming |
| Repeated agent cards | `<template id="agentTpl">` | `cloneNode` + wires events |
| Score bars | width + transition in CSS | sets `style.width = pct%` |
| Show/hide results | the `hidden` attribute | removes/sets `hidden` |

> 💡 **Interview question:** "How did you do responsive and multi-color without a framework?" → CSS Grid `auto-fill` for self-flowing layout, CSS custom properties (`--c`) for dynamic colors set from JS, and `@media` for mobile. All plain CSS, lightweight and build-free.
