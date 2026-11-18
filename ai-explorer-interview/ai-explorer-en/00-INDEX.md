# 📚 Interview Prep Pack — AI Explorer Project

Welcome. This pack takes you **from zero to confident** when interviewing about AI Explorer — an "interactive AI museum" that runs entirely in the browser. Each file digs into one aspect. Read them in order.

---

## 🗺️ Learning path (read in order)

| # | File | Content | Read when |
|---|------|---------|-----------|
| 1 | `01-project-overview.md` | What the project is, what problem it solves, how to demo | First, for the big picture |
| 2 | `02-foundational-concepts.md` | Background theory: ES Modules, PWA, service worker, i18n… **and** every AI concept the rooms illustrate | When a term is unclear |
| 3 | `03-system-architecture.md` | Architecture, hashchange router, the ROOMS array, learning tracks, how to add a room | After grasping the concepts |
| 4 | `04-code-deep-dive.md` | Explains each file (app.js, i18n.js, store.js, roomstate.js, roomquiz.js, sw.js, pwa.js, sound.js…) | To study the code in depth |
| 5 | `05-standout-features.md` | Deep analysis of the "point-scoring" features | To tell an impressive story |
| 6 | `06-interview-questions.md` | 40+ Q&A, easy to hard | To rehearse answers |
| 7 | `07-cheat-sheet.md` | One-page summary for last-minute review | Right before the interview |
| 8 | `08-code-walkthrough.md` | Appendix: the real run of `route()`, a typical room file, and the Reasoning room | To understand the source cold |

---

## ⏱️ If you are short on time

- **1 day left**: files 1 → 2 → 3 → 6 → 7.
- **2 hours left**: files 1 → 3 → 7, then skim file 6.
- **30 minutes left**: file 7 (cheat sheet) and the "most common questions" in file 6.

---

## 🎯 How to use this pack effectively

1. **Understand, don't memorize.** Interviewers ask "why", so know the reasoning behind each decision (why no framework, why ES modules, why cache versioning…).
2. **Re-tell the project in your own words** after files 1 and 3. Fluent in 2 minutes = ready.
3. **Practice answers out loud** for the questions in file 6.
4. **Prepare 2-3 stories**: your hardest technical challenge (e.g. computing WCAG contrast for text, or the search box focus trap), an architecture decision you're proud of (the per-room module architecture), and the **new Reasoning room** (chain-of-thought) as proof you keep up with AI trends.

---

## 📦 The project in 3 sentences (memorize)

> AI Explorer is an **"interactive AI museum"** of **26 rooms**, each explaining one AI concept visually (from neural nets, tokens, embeddings, attention… to RAG, agents, RLHF, and even **chain-of-thought / reasoning models**) and letting the user **play hands-on**. It runs **100% in the browser with vanilla JS + ES Modules — no backend, no API keys** — is installable as an app and **works offline** via a service worker (PWA), fully **bilingual VI/EN**. The architecture is **modular**: each room is a standalone file exporting `render(root)`, adding one entry to the `ROOMS` array in `app.js`; CI runs three test layers (cache integrity, bilingual integrity, Playwright smoke test) and **blocks deploy if any fail**.

Good luck with your interview! 🍀
