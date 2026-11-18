# 📚 Interview Prep Pack — Quán quen (Taste Map) Project

Welcome. This pack takes you **from zero to confident** when interviewing about the Quán quen project. Each file digs into one aspect. Read them in order.

> All docs were written after **directly reading the real source** (app.js 4,718 lines, i18n.js, sw.js, index.html, style.css, the 47-case Playwright test file). No guessing.

---

## 🗺️ Learning path (read in order)

| # | File | Content | Read when |
|---|------|---------|-----------|
| 1 | `01-project-overview.md` | What it is, what it solves, tech stack, how to run | First |
| 2 | `02-foundational-concepts.md` | Theory: PWA, Leaflet, IndexedDB, Web Crypto, map services, i18n, Playwright/CI | When a term is unclear |
| 3 | `03-system-architecture.md` | Architecture, startup flow, render pipeline, data model | After grasping concepts |
| 4 | `04-code-deep-dive.md` | Explains each file (app.js, i18n.js, sw.js, index.html, tests) | To study the code |
| 5 | `05-standout-features.md` | Deep analysis of the "point-scoring" features | To tell an impressive story |
| 6 | `06-interview-questions.md` | 70+ Q&A | To rehearse answers |
| 7 | `07-cheat-sheet.md` | One-page summary for go-time | Last minute |
| 8 | `08-code-walkthrough.md` | Appendix: every function in app.js | To understand 100% of the source |

---

## ⏱️ If you are short on time

- **1 day left**: files 1 → 2 → 3 → 6 → 7.
- **2 hours left**: files 1 → 3 → 7, then skim 6.
- **30 minutes left**: file 7 + the "most common questions" in file 6.

---

## 📦 The project in 3 sentences (memorize)

> Quán quen is a **PWA web app** to save and manage food/cafe spots **by personal taste** on a map — something Google Maps can't do: rate by your own criteria (tasty, quiet, power outlets, good for dates, good for working), suggest "where to go today" by mood, plan multi-stop itineraries with real routes, and discover new places around you from OpenStreetMap. It's all **plain HTML/CSS/JS (no framework, no bundler)** using **Leaflet** for maps; **no backend** — data lives locally in **localStorage + IndexedDB** (images), with optional sync via **GitHub Gist** and **AES-GCM encrypted backups**. It has **47 automated Playwright tests + GitHub Actions CI/CD** (deploys only when tests pass), is bilingual VI/EN, and works offline.

Good luck with your interview! 🍀
