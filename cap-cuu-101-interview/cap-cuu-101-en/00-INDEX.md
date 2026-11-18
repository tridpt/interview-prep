# 📚 Interview Prep Pack — Cấp Cứu 101 Project

Welcome. This pack takes you **from zero to confident** when interviewing about the Cấp Cứu 101 (First Aid 101: "Don't Die Dumb") project. Each file digs into one aspect. Read them in order.

---

## 🗺️ Learning path (read in order)

| # | File | Content | Read when |
|---|------|---------|-----------|
| 1 | `01-project-overview.md` | What the project is, what problem it solves, how to demo | First, for the big picture |
| 2 | `02-foundational-concepts.md` | Background theory: PWA, service worker, Web APIs, i18n... | When a term is unclear |
| 3 | `03-system-architecture.md` | Architecture, startup flow, router, data model | After grasping the concepts |
| 4 | `04-code-deep-dive.md` | Explains each file (app.js, sound.js, sw.js, index.html...) | To study the code in depth |
| 5 | `05-standout-features.md` | Deep analysis of the "point-scoring" features | To tell an impressive story |
| 6 | `06-interview-questions.md` | 60+ Q&A, easy to hard | To rehearse answers |
| 7 | `07-cheat-sheet.md` | One-page summary for last-minute review | Right before the interview |
| 8 | `08-code-walkthrough.md` | Appendix: walking through EVERY function in app.js (2,864 lines) | To understand 100% of the source |

---

## ⏱️ If you are short on time

- **1 day left**: files 1 → 2 → 3 → 6 → 7.
- **2 hours left**: files 1 → 3 → 7, then skim file 6.
- **30 minutes left**: file 7 (cheat sheet) and the "most common questions" in file 6.

---

## 🎯 How to use this pack effectively

1. **Understand, don't memorize.** Interviewers ask "why", so know the reasoning behind each decision.
2. **Re-tell the project in your own words** after files 1 and 3. Fluent in 2 minutes = ready.
3. **Practice answers out loud** for the questions in file 6.
4. **Prepare 2-3 stories**: your hardest challenge, a technical decision you're proud of, and how you handled the "medical app but fun" tension.

---

## 📦 The project in 3 sentences (memorize)

> Cấp Cứu 101 is a **PWA web app** that teaches life-saving first aid (CPR, choking, burns, bleeding control...) through **funny cartoon mini-games** in the style of *Dumb Ways to Die*, plus a **Panic Mode** that reads each step aloud for real emergencies. The whole thing is **plain HTML/CSS/JS — no framework, no build, no external libraries** — and it **runs 100% offline** via a service worker, which is critical for an emergency app. It uses many browser Web APIs (Web Audio for synthesized sound, Web Speech for read-aloud, Vibration, DeviceOrientation, Notifications, Web Share), supports **Vietnamese/English bilingual**, and stores all data locally with localStorage (nothing is sent anywhere).

Good luck with your interview! 🍀
