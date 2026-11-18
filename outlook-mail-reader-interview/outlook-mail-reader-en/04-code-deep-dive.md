# 4. Code Deep Dive (file by file)

Read alongside the real code in `outlook-mail-reader/`. For a function-by-function walk of `engine.py`, see `08-code-walkthrough.md`.

---

## 4.1. `config.py` — Configuration

- `BASE_DIR` / `STORAGE_DIR` (auto-created) and three file paths: `TOKEN_CACHE_PATH`, `ENCRYPTION_KEY_PATH` (`cache.key`), `IMPORTED_ACCOUNTS_PATH`.
- `CLIENT_ID = os.environ.get("OUTLOOK_CLIENT_ID", "")` — from an env var.
- `AUTHORITY = ".../common"` (personal + org).
- `SCOPES = ["Mail.ReadWrite", "User.Read"]` (read + mark-as-read, **no send**).
- `GRAPH_BASE`, `TOKEN_ENDPOINT`, `IMAP_SCOPES` (`IMAP.AccessAsUser.All` + `offline_access`), `IMAP_HOST/PORT` (`outlook.office365.com:993`).
- `DEFAULT_MAIL_COUNT = 15`.

> 💡 The file's comments state the philosophy: **no passwords stored**, log in once per account, then MSAL auto-uses the refresh token.

---

## 4.2. `store.py` — Encrypted token storage

**Job:** serialize the MSAL cache (holding refresh tokens for many accounts) then **Fernet-encrypt** before writing to disk.

- `_load_or_create_key()`: read `cache.key` or `Fernet.generate_key()` then `os.chmod(..., S_IRUSR|S_IWUSR)` (600).
- `load_cache()`: a `SerializableTokenCache`; if the file exists → `Fernet(key).decrypt(...)` → `cache.deserialize(...)`; `except Exception: pass` (corrupt → skip).
- `save_cache(cache)`: `if not cache.has_state_changed: return` → `fernet.encrypt(cache.serialize().encode())` → write → chmod 600.
- `load_imported_accounts()` / `save_imported_accounts()`: same for the imported-accounts file (JSON ↔ Fernet), returning a safe `{"version":1,"accounts":{}}` default on error/empty.

> 💡 All read operations are wrapped in try/except returning safe defaults → a corrupt cache won't crash the app.

---

## 4.3. `engine.py` — The heart (1,524 lines)

`OutlookEngine` (singleton `engine` at the end). Init: load cache + imported accounts, a `threading.Lock`, dicts `_login_jobs` / `_import_jobs`. Groups (details in file 08):

### Login & tokens
- `_build_app()`: create `msal.PublicClientApplication(CLIENT_ID, authority, token_cache)`; error if CLIENT_ID missing.
- `begin_login()` / `login_status()`: device code flow + background thread + job tracking.
- `_token_for(home_account_id)`: get an access token — imported graph → `_refresh_imported_access_token`; MSAL → `acquire_token_silent`. The **token entry point for every Graph call**.

### Refresh-token import
- `_parse_import_line`: split `email|password|refresh_token|client_id` (exactly 4 parts, validate email/token/client_id), **drops the password**.
- `_redeem_refresh_token`: POST `TOKEN_ENDPOINT` to exchange a refresh token for an access token.
- `_graph_import_scope_options` + `_try_graph_import`: **probe scopes** (try Mail.Read/Mail.ReadWrite... until `GET /me/messages?$top=1` returns 200) to learn read/write permission.
- `import_refresh_token_account`: try Graph first, else **fallback IMAP** (`_imap_probe`); store the account encrypted.
- `import_refresh_token_lines` (serial) and `begin_import_refresh_token_job` (**multithreaded**: Queue + N workers + Lock + progress/cancel) + `import_job_status` (paginated log) + `cancel_import_job`.

### Account management & health
- `list_accounts`: merge MSAL accounts (`app.get_accounts()`) + imported.
- `remove_account`: delete from imported or from MSAL.
- `check_account_health`: IMAP opens INBOX or Graph reads 1 mail; classifies errors (`_classify_*`: token_expired/missing_scope/...).
- `account_unread_counts`: per-account inbox unread count (for badges).

### Reading mail (Graph + IMAP in parallel)
- `get_messages` / `get_unified_inbox` / `get_message_detail` / `search_messages` / `mark_read` / `list_attachments` / `get_attachment` / `list_folders` — each **checks the mode** then branches Graph or IMAP.
- `_build_filter`: assemble the OData `$filter` (isRead/hasAttachments/date). Sender filtering is done **client-side** (Graph can't fuzzy-match sender address).
- The `_imap_*` block: XOAUTH2 connect, SELECT/SEARCH/FETCH, parse `email`/`BytesParser`, `_extract_body` (prefer HTML), encode/decode IMAP mail id (`imap:<folder_b64>:<uid>`).

### Email check
- `check_email_addresses` + `_resolve_mx` (Cloudflare DoH) + `_infer_email_provider` + `_valid_email_address`.

---

## 4.4. `routes.py` — API (~210 lines)

- `APIRouter(prefix="/api/outlook")`. Pydantic models: `ImportRefreshTokenRequest`, `UpdateRefreshTokenRequest`, `CheckEmailsRequest`.
- `FOLDER_MAP`: friendly names (inbox/sent/...) → Graph well-known folders (sentitems/junkemail/...).
- Every route **lazy-imports** `from engine import engine` then calls the relevant method; catches `RuntimeError` → `HTTPException` (400/404).
- The attachment-download endpoint returns a `Response` with `Content-Disposition: attachment; filename*=UTF-8''<quote>` (supports Unicode filenames).

> 💡 **Why lazy-import engine in each route?** To avoid initializing `OutlookEngine()` (file reads, MSAL setup) too early at module import, and to avoid import cycles. Engine is a singleton, so importing many times still shares one instance.

---

## 4.5. `main.py` — The app

- `FastAPI(...)` + `CORSMiddleware(allow_origins=["*"], ...)`.
- `include_router(outlook_router)`.
- `/api/health` → `{status, configured: bool(CLIENT_ID)}`.
- `app.mount("/", StaticFiles(directory=frontend, html=True))` — serves the frontend (mounted **last** so it doesn't shadow `/api` routes).

---

## 4.6. `frontend/index.html` + `app.js` (~810 lines)

- **index.html**: cards (accounts, email check, inbox), folder tabs, filters, two modals (change token, mail detail). The mail body is an **`<iframe id="m-body" sandbox>`**.
- **app.js** (vanilla JS):
  - `api(path, opts)`: wraps `fetch`, extracts the error `detail` from JSON.
  - `state`: folder, searchMode, order/loaded/pages/hasMore (per-account pagination), accounts, importJob.
  - **Account**: `loadStatus`, `renderAccounts` (table), `checkAccountHealth`, `openTokenModal`/`saveTokenUpdate`, `loadUnreadCounts` (badge).
  - **Import**: `startImportLines` → create job → `pollImportJob` (1.5s) → render progress/log/pager; `copyFailedLines`/`retryFailedLines`/`cancelImportJob`.
  - **Login**: `startLogin` → show code/link → `pollLogin` (2.5s).
  - **Inbox**: `loadInbox`/`loadMore`/`renderInbox`/`mailItemHtml`; filters via `filterParams()`; folder tabs.
  - **Search**: `doSearch`/`clearSearch`.
  - **Mail detail**: `openMail` sets `iframe.srcdoc = body` (HTML) or `<pre>` escaped (text); auto `setRead(true)` on open; `loadAttachments`; read toggle.
  - **Email check**: `checkEmailsLive` + a results table.

> 💡 **Frontend security point:** the HTML body is set into `iframe[sandbox].srcdoc` → scripts in the mail **don't run**. All other dynamic strings go through `escapeHtml`.

---

## 4.7. Summary of technical decisions

| Decision | Reason |
|----------|--------|
| OAuth 2.0 + MSAL, no passwords | Standard, secure; MSAL auto-refreshes + multi-account cache |
| Device code flow | Fits an app with no web redirect; simple code-entry UX |
| Fernet token encryption | Don't keep refresh tokens plaintext on disk |
| Graph + IMAP abstraction | Support both Graph tokens and IMAP-only tokens behind one API |
| Scope probing on import | Auto-learn whether a token has Mail.Read or ReadWrite |
| Multithreaded import job (Queue+Lock) | Speed up bulk import, with progress/cancel |
| Per-account error isolation | A dead account doesn't take down the unified inbox |
| iframe sandbox for mail HTML | Prevent XSS/scripts/trackers in mail |
| Lazy-import engine in routes | Avoid import cycles + early init |
| Email check via DNS/MX (DoH) | Check a domain can receive mail without sending a test |
