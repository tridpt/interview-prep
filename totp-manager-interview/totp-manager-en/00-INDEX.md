# 📚 Interview Prep Pack — OTP Vault (totp-manager) Project

Welcome. This pack takes you **from zero to confident** about OTP Vault — a 2FA (TOTP) manager. Each file digs into one aspect. Read in order.

> All docs were written after **directly reading 100% of the real source**: backend (`config.py`, `totp.py`, `store.py`, `main.py`), the test suite (RFC 6238 vectors + property-based hypothesis + FastAPI TestClient), and frontend (`app.js`, `index.html`, `sw.js`). No guessing.

---

## 🗺️ Learning path (read in order)

| # | File | Content | Read when |
|---|------|---------|-----------|
| 1 | `01-project-overview.md` | What it is, what it solves, tech stack, how to run | First |
| 2 | `02-foundational-concepts.md` | Theory: TOTP/RFC 6238, HMAC, base32, scrypt, Fernet/AES, FastAPI, PWA, property-based testing | When a term is unclear |
| 3 | `03-system-architecture.md` | Architecture, data flow, security model, session lifecycle | After grasping concepts |
| 4 | `04-backend-deep-dive.md` | Backend file by file, function by function | To study the server |
| 5 | `05-frontend-deep-dive.md` | app.js, index.html, sw.js | To study the client |
| 6 | `06-standout-features.md` | Deep analysis of the "point-scoring" features | To tell an impressive story |
| 7 | `07-interview-questions.md` | 70+ Q&A | To rehearse answers |
| 8 | `08-cheat-sheet.md` | One-page summary for go-time | Last minute |

---

## ⏱️ If you are short on time

- **1 day left**: files 1 → 2 → 3 → 7 → 8.
- **2 hours left**: files 1 → 3 → 8, then skim 7.
- **30 minutes left**: file 8 + the "most common questions" in file 7.

---

## 📦 The project in 3 sentences (memorize)

> OTP Vault is a locally-run web app for **managing 2FA (TOTP) codes** — paste a secret key, get a 6-digit code that changes every 30 seconds, same standard as Google Authenticator/Authy (**RFC 6238**). All accounts are stored in an **encrypted vault** (`vault.bin`) protected by a **master password**: the encryption key is derived from the password via **scrypt** and **never written to disk** — forgetting the password means losing the data (by design). The backend is **FastAPI (Python)** using the standard `hmac`/`hashlib` for TOTP and **Fernet (AES) + scrypt** for the vault; the frontend is **plain HTML/CSS/JS (PWA)** with QR scanning (jsQR), drag-to-reorder, and backup/restore. The project has a **rigorous test suite** with RFC 6238 test vectors and **property-based testing (hypothesis)**.

Good luck with your interview! 🍀
