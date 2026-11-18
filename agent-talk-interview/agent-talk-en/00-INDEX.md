# 📚 Interview Prep Pack — Agent Talk Project

Welcome. This pack is built to take you **from zero to confident** when interviewing about the Agent Talk project. Each file digs into one aspect. Read them in the order below.

---

## 🗺️ Learning path (read in order)

| # | File | Content | Read when |
|---|------|---------|-----------|
| 1 | `01-project-overview.md` | What the project is, what problem it addresses, how to demo | First, to get the big picture |
| 2 | `02-foundational-concepts.md` | All the background theory: LLM, API, streaming, async, FastAPI... | When a term is unclear |
| 3 | `03-system-architecture.md` | Overall architecture, data flow, diagrams | After grasping the concepts |
| 4 | `04-backend-deep-dive.md` | Explains each backend file/endpoint | To study the server in depth |
| 5 | `05-frontend-deep-dive.md` | Explains frontend logic (conversation loop, session save...) | To study the client in depth |
| 6 | `06-standout-features.md` | Deep analysis of the "point-scoring" features | To tell an impressive story |
| 7 | `07-interview-questions.md` | 60+ Q&A, from easy to hard | To rehearse answers |
| 8 | `08-cheat-sheet.md` | One-page summary for last-minute review | Right before the interview |
| 9 | `09-frontend-html-css.md` | Appendix: reading index.html + style.css directly (every id/class) | To understand 100% of every file |

---

## ⏱️ If you are short on time

- **1 day left**: Read files 1 → 2 → 3 → 7 → 8.
- **2 hours left**: Read files 1 → 3 → 8, then skim file 7.
- **30 minutes left**: Read file 8 (cheat sheet) and the "most common questions" section in file 7.

---

## 🎯 How to use this pack effectively

1. **Understand, don't memorize.** Interviewers ask "why", so you need the reasoning behind each decision.
2. **Re-tell the project in your own words** after finishing files 1 and 3. If you can do it fluently in 2 minutes, you're ready.
3. **Practice answers out loud** for the questions in file 7. Answering in your head is very different from speaking.
4. **Prepare 2-3 stories** about your "hardest challenge" and a "technical decision you're proud of" (hints in files 6 and 7).

---

## 📦 The project in 3 sentences (memorize this)

> Agent Talk is a web app where multiple "AI characters" with distinct personalities talk and debate with each other in real time, while the user watches and can step in to steer the conversation. The backend uses **FastAPI (Python)** as an orchestrator: it calls Large Language Models (LLMs) through an OpenAI-compatible API, supports **token-by-token streaming**, summarizes long conversations, does web lookups, scores debates, and keeps long-term memory. The frontend is **plain HTML/CSS/JavaScript** (no framework) that manages the turn-taking loop, saves sessions in localStorage, and exports files.

Good luck with your interview! 🍀
