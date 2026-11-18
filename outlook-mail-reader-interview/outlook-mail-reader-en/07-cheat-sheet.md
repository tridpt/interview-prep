# 7. Cheat Sheet — Quick Review Before Go-Time

Skim for 5-10 minutes before the interview.

---

## 🎯 30-second pitch (memorize)
> Outlook Mail Reader merges **multiple Outlook/M365 inboxes** into one place. Each account **logs in via OAuth once** (device code flow), then the app auto-uses a **refresh token** (stored **encrypted**) to fetch mail without re-login — **no passwords stored**. Backend **FastAPI + MSAL + Microsoft Graph** (with **IMAP XOAUTH2** fallback): unified inbox, search, mark-as-read, attachments, **multithreaded token import** with progress/cancel, email check via **DNS/MX**. Frontend **vanilla JS**, renders mail HTML in a **sandboxed iframe**.

## 🧱 Tech stack
- **Backend**: FastAPI + Uvicorn, **MSAL** (OAuth/token), **requests** (Graph + DoH), **cryptography/Fernet** (encryption), **imaplib** (IMAP), **threading/queue** (jobs).
- **Frontend**: vanilla HTML/CSS/JS; **iframe sandbox** for mail HTML.
- **Services**: login.microsoftonline.com (OAuth), graph.microsoft.com, outlook.office365.com (IMAP), cloudflare-dns.com (MX).
- **Config**: `OUTLOOK_CLIENT_ID` (Azure App, public client).

## 🗂️ Files
| File | Role |
|---|---|
| config.py | client_id, scope, endpoints, paths |
| store.py | Fernet-encrypt token cache + imported accounts |
| engine.py (1524) | OutlookEngine: OAuth, tokens, Graph+IMAP, import job, email check |
| routes.py | API /api/outlook/* (lazy-import engine) |
| main.py | FastAPI + CORS + mount frontend |
| frontend/app.js (810) | UI: login poll, inbox, search, import job, iframe sandbox |

## 🔑 OAuth (KNOW IT)
- **Access token**: short-lived, API calls. **Refresh token**: long-lived, exchanges for new access (stored encrypted).
- **Scope**: Mail.ReadWrite + User.Read + offline_access (to get a refresh token).
- **Device code flow**: app gives code + link → user enters in a browser → app polls until done.
- **MSAL**: PublicClientApplication, `acquire_token_by_device_flow`, **`acquire_token_silent`** (auto-refresh), multi-account token cache.
- **Public client** + "Allow public client flows" = Yes in Azure.

## 🔐 Encrypted token storage
serialize MSAL cache → **Fernet (AES) encrypt** → token_cache.bin; key at cache.key (chmod 600); write when `has_state_changed`. **Limit**: key on same machine → prevents accidental exposure, not a machine-owning attacker.

## 🔀 Two auth modes
- **graph (MSAL)**: device login → silent refresh.
- **imported**: paste refresh token → `_redeem_refresh_token` + **scope probing** → Graph; if not → **IMAP** (XOAUTH2 + INBOX probe).
- Every mail operation checks `mode` then branches Graph/IMAP.

## 🌐 Graph + IMAP
- Graph: REST + Bearer; `/me/messages` ($top/$skip/$orderby/$select/$filter), `$search`+ConsistencyLevel, PATCH isRead, /attachments, /mailFolders.
- IMAP: `IMAP4_SSL outlook.office365.com:993`, **XOAUTH2** (`user=..\x01auth=Bearer ..`), SELECT/UID SEARCH/UID FETCH, FLAGS `\Seen`, mail id = `imap:<b64folder>:<uid>`.

## 🧵 Multithreaded import job
Queue + 1-5 worker threads + **threading.Lock** (race-safe) → progress % + paginated log + cancel (cancel_requested flag) + copy/retry failed lines. Login also uses a **background thread + polling**.

## 🛡️ Security
1. OAuth, **no passwords** (the import-line password is dropped).
2. Tokens **Fernet-encrypted**.
3. Mail HTML in an **iframe sandbox** (no script execution); escapeHtml for dynamic strings.
4. Mark-read blocked if the token is Mail.Read-only.

## 📨 Email check
format + **MX via Cloudflare DoH** + provider guess + IDN. Does **not** verify the specific mailbox.

## 🧱 Architecture
config ← store ← engine (singleton, Lock) ← routes (lazy import) ← main. Backend **stateful** (token files + RAM jobs). Per-account error isolation.

## ⚖️ Trade-offs / limits
Key on same machine; CORS `*` (local use); large engine.py (1,524 lines); unified inbox is serial (slow with many accounts); jobs in RAM lost on restart. Token import needs the "only accounts you own" constraint.

## 🚀 Future improvements
Key from OS keychain; API auth + tighter CORS; parallelize the unified inbox; pool IMAP; split engine.py; compose/send mail, new-mail notifications, TTS.

## 💬 Key terms
- **OAuth 2.0 / access token / refresh token / scope / offline_access**
- **MSAL / device code flow / acquire_token_silent / public client**
- **Microsoft Graph / OData ($filter/$search) / Bearer**
- **IMAP / XOAUTH2 / UID / FLAGS \Seen**
- **Fernet/AES / cache.key / chmod 600**
- **threading / Queue / Lock / daemon / polling**
- **iframe sandbox / escapeHtml / XSS**
- **DNS-over-HTTPS / MX record / IDN**

---

### ✅ Pre-interview checklist
- [ ] 30-second pitch.
- [ ] Access vs refresh token + device code flow.
- [ ] MSAL & why not hand-roll OAuth.
- [ ] Encrypted token storage + limit (key on same machine).
- [ ] Why both Graph and IMAP (adapter by mode).
- [ ] Multithreaded import job (Queue/Lock/cancel).
- [ ] iframe sandbox safe HTML rendering.
- [ ] 2 limitations + fixes.
- [ ] 2 questions for the interviewer.

**Breathe. You understand this project. Tell your story with confidence. 🍀**
