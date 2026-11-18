# 3. System Architecture

## 3.1. High-level diagram

```
┌───────────────────────── BROWSER (frontend) ─────────────────────────────┐
│  index.html + style.css + app.js (vanilla JS)                             │
│  - device login (poll), unified inbox, search, filters, import job UI     │
│  - render mail HTML in <iframe sandbox>                                   │
└───────────────────────────────┬───────────────────────────────────────────┘
                                │ HTTP (fetch) /api/outlook/*
                                ▼
┌──────────────────────── BACKEND — FastAPI (main.py) ──────────────────────┐
│  routes.py  — endpoints, validation (Pydantic), lazy import engine        │
│       │                                                                    │
│       ▼                                                                    │
│  engine.py — OutlookEngine (singleton): login, tokens, Graph/IMAP,        │
│              multithreaded import job, email check, sanitize              │
│       │                  │                         │                       │
│       ▼                  ▼                         ▼                       │
│  store.py (encrypt)  MSAL (token cache)      requests / imaplib            │
│  Fernet/AES          PublicClientApp                                       │
└───────┬──────────────────┬─────────────────────────┬──────────────────────┘
        │                  │                          │
        ▼                  ▼                          ▼
  storage/*.bin     login.microsoftonline.com   graph.microsoft.com
  (encrypted token) (OAuth)                      outlook.office365.com (IMAP)
                                                 cloudflare-dns.com (MX)
```

**Core trait:** this is a **backend with real identity/OAuth integration**, not just CRUD. `engine.py` is the heart (1,524 lines) — handling tokens, Graph/IMAP calls, multithreading. The backend **holds state** (token cache, login jobs, import jobs in RAM + encrypted files) — unlike client-only static apps.

## 3.2. Layers & file responsibilities

| File | Role |
|------|------|
| `config.py` | Config constants: `CLIENT_ID`, `AUTHORITY`, `SCOPES`, Graph/IMAP/token endpoints, storage paths |
| `store.py` | Fernet encryption: load/save the MSAL token cache + imported accounts; key management |
| `engine.py` | `OutlookEngine`: login, token refresh, Graph + IMAP, multithreaded import job, email check, sanitize. **Singleton `engine`** |
| `routes.py` | `APIRouter` `/api/outlook/*`: receive request, validate, call engine, map errors → HTTP |
| `main.py` | Create the FastAPI app, CORS, include router, `/api/health`, mount the static frontend |
| `frontend/*` | UI: call API, render inbox/mail, manage import job, iframe sandbox |

## 3.3. The two auth modes (the most important part)

A distinctive architectural point — each account is in one `mode`:

### Mode "graph" via MSAL (interactive login)
- The user logs in via **device code flow**.
- MSAL stores the refresh token in a **shared token cache** (encrypted via `store.py`).
- Getting a token later: `acquire_token_silent()` (MSAL auto-refreshes).
- `home_account_id` issued by MSAL.

### Mode "imported" (paste an existing refresh token)
- The user pastes `email|password|refresh_token|client_id` (password is **ignored**).
- The engine **redeems the refresh token** for an access token (`_redeem_refresh_token` → POST `TOKEN_ENDPOINT`), **trying multiple scope sets** (`_graph_import_scope_options`: Mail.Read/Mail.ReadWrite...) to probe permissions.
- If Graph doesn't work → try **IMAP** (mode "imap"): redeem IMAP scopes then `_imap_probe`.
- Stored in `imported_accounts.bin` (separately encrypted), `home_account_id = imported:<sha256>`.

➡️ **Every mail operation** (`get_messages`, `get_message_detail`, `search`, `mark_read`, attachments, folders, unread count) **checks the mode** then branches to Graph or IMAP. A **two-backend abstraction** under one API.

## 3.4. Login flow (device code) — study this

```
1. [Frontend] click "+ Add account" → POST /login/start
2. [Engine] begin_login(): MSAL initiate_device_flow → {user_code, verification_uri}
   + create job_id + run a BACKGROUND THREAD: acquire_token_by_device_flow (blocks until done)
3. [Frontend] show code + link; poll GET /login/status/{job_id} every 2.5s
4. [User] open the link, enter the code, log into the real Microsoft
5. [Background thread] receive token → save_cache() (encrypt) → job.status = "success"
6. [Frontend] sees "success" → hide the login area, reload the account list
```

## 3.5. Unified inbox flow

```
[Frontend] GET /inbox?folder=&count=&skip=&unread_only=&sender=&date_from=...&account=
   │
[routes] map the friendly folder → Graph well-known folder (FOLDER_MAP)
   ▼
[engine.get_unified_inbox] loop each account:
   - graph → get_messages() builds a Graph URL ($top/$skip/$orderby/$select/$filter)
   - imap  → _imap_get_messages() (SELECT + UID SEARCH + UID FETCH + parse)
   - one account errors → record "error" for that account, does NOT break the response
   ▼
return [{account, home_account_id, messages, has_more, error?}]
```
Each account is wrapped in its own try/except → **a dead account doesn't take down the whole inbox**.

## 3.6. Multithreaded import job

```
POST /accounts/import-refresh-token/job {line: "...many lines...", concurrency: 1-5}
   ▼
begin_import_refresh_token_job:
   - push every line into a Queue
   - start N daemon worker threads; each: take a line → import_refresh_token_account → record result
   - a Lock protects job state (processed/added/failed/current_lines)
   - the last worker sets status = done/cancelled
   ▼
[Frontend] poll GET /import-jobs/{id}?offset=&limit= → progress % + paginated log
           with a Cancel button (POST .../cancel → sets cancel_requested)
```

## 3.7. Where does the backend state live?

- **Persistent (encrypted files)**: `token_cache.bin` (MSAL cache), `imported_accounts.bin`, `cache.key`.
- **Transient (RAM)**: `_login_jobs`, `_import_jobs` (dicts in the engine singleton) — lost on server restart.
- ➡️ The backend is **stateful** (unlike client-only apps). A single shared `engine` instance, with a `threading.Lock` for thread safety.

## 3.8. Why is this architecture sound?

- **Clear layering**: config ← store (encryption) ← engine (logic+network) ← routes (HTTP) ← frontend.
- **Two-backend abstraction** (Graph/IMAP) behind one unified API → the frontend needn't know which mechanism an account uses.
- **Error isolation**: each account/import line wrapped in try/except → local errors don't spread.
- **Layered security**: OAuth (no passwords) + encrypted tokens + sandboxed iframe for mail HTML.
