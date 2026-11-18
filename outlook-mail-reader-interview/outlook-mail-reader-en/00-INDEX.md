# 📚 Interview Prep Pack — Outlook Mail Reader Project

Welcome. This pack takes you **from zero to confident** when interviewing about the Outlook Mail Reader (multi-account inbox) project. Read in order.

> All written after **directly reading 100% of the real source**: `config.py`, `main.py`, `store.py`, `engine.py` (1,524 lines), `routes.py`, `frontend/index.html`, `frontend/app.js` (810 lines).

---

## 🗺️ Learning path (read in order)

| # | File | Content | Read when |
|---|------|---------|-----------|
| 1 | `01-project-overview.md` | What it is, what it solves, tech stack, how to run | First |
| 2 | `02-foundational-concepts.md` | Theory: OAuth 2.0, MSAL, device code flow, refresh token, Graph API, IMAP XOAUTH2, Fernet encryption, threading, DNS-over-HTTPS, iframe sandbox | When a term is unclear |
| 3 | `03-system-architecture.md` | Architecture, login flow, the two auth modes, data flow | After grasping concepts |
| 4 | `04-code-deep-dive.md` | Explains each file (config/main/store/engine/routes/frontend) | To study the code |
| 5 | `05-standout-features.md` | Deep analysis of the "point-scoring" features | To tell an impressive story |
| 6 | `06-interview-questions.md` | 70+ Q&A | To rehearse answers |
| 7 | `07-cheat-sheet.md` | One-page summary for go-time | Last minute |
| 8 | `08-code-walkthrough.md` | Appendix: every function in engine.py | To understand 100% of the source |

---

## ⏱️ If you are short on time

- **1 day left**: files 1 → 2 → 3 → 6 → 7.
- **2 hours left**: files 1 → 3 → 7, then skim 6.
- **30 minutes left**: file 7 + the "most common questions" in file 6.

---

## 📦 The project in 3 sentences (memorize)

> Outlook Mail Reader is a **web app that merges the inboxes of multiple Outlook/Microsoft 365 accounts** into one place: each account logs in via **OAuth once** (device code flow), then the app uses a **refresh token** (stored **encrypted** on disk) to fetch mail without logging in again — and it **never stores passwords**. The backend is **FastAPI (Python)** using **MSAL + Microsoft Graph API** (with **IMAP XOAUTH2** as a fallback), supporting a unified inbox, search, mark-as-read, attachments, **bulk refresh-token import via a multi-threaded job with progress/cancel**, and email checking via **DNS/MX** lookup. The frontend is **vanilla JS** that renders mail HTML inside a safe **sandboxed iframe**.

Good luck with your interview! 🍀
