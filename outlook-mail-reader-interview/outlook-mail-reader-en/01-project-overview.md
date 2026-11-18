# 1. Project Overview — Outlook Mail Reader

## 1.1. What is Outlook Mail Reader?

Outlook Mail Reader is a **mail-reading web app** that merges the mail of **multiple Outlook / Microsoft 365 accounts** into a single interface (unified inbox). The core idea: each account logs into Microsoft **exactly once**; afterward the app uses a **refresh token** (stored encrypted) to fetch mail without asking to log in again, and it **never stores passwords**.

### Core points to remember
- **Multi-account**: merges many inboxes into one place.
- **Microsoft-standard OAuth 2.0** (Microsoft Graph) — log in once through the real Microsoft screen.
- **No passwords stored**; refresh tokens stored **encrypted** (Fernet/AES) on disk.
- **Two auth modes**: (1) interactive login (device code flow) via MSAL; (2) importing an existing refresh token (Graph or IMAP).
- Backend **FastAPI**, frontend **vanilla JS**; mail HTML rendered in a **sandboxed iframe**.

## 1.2. What problem does it solve?

1. **Juggling many Outlook tabs/logins** → merge everything into one inbox.
2. **Re-logging in is annoying** → log in once, then auto-refresh the token.
3. **Security** → standard OAuth, no passwords, encrypted tokens.
4. **Managing batches of accounts** → bulk refresh-token import, a multi-threaded job with progress/cancel, token health checks, DNS/MX email checking.
5. **Demonstrating technical ability**: real OAuth/Graph integration, encryption, multithreading, IMAP fallback, safe HTML rendering — the point-scorer.

> 💡 **Interview tip:** this is a project with **real OAuth/identity integration** — highly valuable since nearly every enterprise product needs "Sign in with Microsoft/Google". Emphasize that you understand the **OAuth 2.0 flow, refresh tokens, and secure token storage** — core knowledge of modern backends.

## 1.3. Main features

| Group | Features |
|-------|----------|
| **Login** | Device code flow (open link, enter code), background thread + status polling |
| **Unified inbox** | Merge mail from many accounts; pagination, filters (unread/attachments/sender/date/account), by folder (inbox/sent/drafts/junk/deleted/archive) |
| **View mail** | Full content, HTML rendered in a sandboxed iframe, list + download attachments |
| **Mark** | Read / unread (needs Mail.ReadWrite); unread-count badge per account |
| **Search** | Full-text across all accounts at once (Graph `$search` / IMAP TEXT) |
| **Account mgmt** | View source (Graph/IMAP), scope, token update time; change token; delete; health check |
| **Token import** | Paste many lines `email|password|refresh_token|client_id`; password is ignored, only the encrypted token is stored |
| **Import job** | Background 1-5 threads, progress %, paginated log, cancel job, copy/retry failed lines |
| **Email check** | Validate format + DNS/MX + guess provider (no specific mailbox verification) |

## 1.4. Tech stack

**Backend (Python):**
- **FastAPI** `0.115.0` — web framework, REST API.
- **Uvicorn** `[standard] 0.30.6` — ASGI server.
- **MSAL** `>=1.31.0` (Microsoft Authentication Library) — OAuth/login + multi-account token cache.
- **requests** `>=2.32.0` — calls Microsoft Graph API and DNS-over-HTTPS.
- **cryptography** `>=43.0.0` — **Fernet** (AES) to encrypt the token cache on disk.
- **imaplib** (stdlib) — IMAP XOAUTH2 fallback.
- **threading / queue** (stdlib) — multi-threaded import job.

**Frontend:**
- **Plain HTML + CSS + JavaScript** (no framework, no build).
- **iframe `sandbox`** to render mail HTML safely; `escapeHtml` for dynamic content.

**External services:**
- **Microsoft identity platform** (`login.microsoftonline.com`) — OAuth.
- **Microsoft Graph** (`graph.microsoft.com/v1.0`) — reading mail.
- **Outlook IMAP** (`outlook.office365.com:993`) — fallback.
- **Cloudflare DNS-over-HTTPS** (`cloudflare-dns.com/dns-query`) — MX lookup for email checks.

**Config:** `OUTLOOK_CLIENT_ID` (Azure App Registration, public client) via environment variable.

> 💡 **Question: "Why MSAL instead of writing OAuth yourself?"** → MSAL is Microsoft's official library; it handles the hard parts: device code flow, **auto token refresh**, **multi-account token cache** management, expiry handling. Hand-rolling OAuth is error-prone and less secure. I only added encryption of the cache when writing to disk.

## 1.5. How to run

```bash
cd outlook-mail-reader
python -m venv .venv
.venv\Scripts\activate                     # Windows
set OUTLOOK_CLIENT_ID=<your-client-id>      # client id from the Azure App Registration
pip install -r requirements.txt
cd backend
uvicorn main:app --reload --host 0.0.0.0 --port 8809
```
Open `http://localhost:8809`. On Windows, `start_outlook_reader.bat` auto-creates the venv + installs + sets the client id + runs.

**Azure setup (once):** portal.azure.com → App registrations → New registration (Supported account types = *any org directory + personal Microsoft accounts*) → Authentication → enable **Allow public client flows** = Yes → copy the **Application (client) ID**.

## 1.6. Folder structure

```
outlook-mail-reader/
├── backend/
│   ├── config.py    # client_id, scope, paths, endpoints
│   ├── store.py     # encrypted token cache (Fernet/AES)
│   ├── engine.py    # device-code login, auto refresh, Graph/IMAP calls (1,524 lines)
│   ├── routes.py    # API /api/outlook/*
│   └── main.py      # FastAPI app + serve frontend
├── frontend/        # index.html, style.css, app.js
└── storage/         # token_cache.bin, imported_accounts.bin, cache.key (all encrypted/secret)
```

> ⚠️ **Ethical & legal note (stated by the project):** It uses Microsoft-standard OAuth, **stores no passwords**, and each account still goes through the real Microsoft login once — so it should **only be used with accounts you own or are authorized to access**. The requested scope is `Mail.ReadWrite` (read + mark-as-read, **no send/delete**). In interviews, present it as a **personal multi-account mail client** and emphasize the OAuth/security engineering.
