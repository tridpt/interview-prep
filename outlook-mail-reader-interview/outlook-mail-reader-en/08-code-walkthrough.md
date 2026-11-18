# 8. Appendix: Code Walkthrough

> ⚠️ Written after **directly reading 100% of the source**: `config.py`, `store.py`, `engine.py` (1,524 lines — fully read), `routes.py`, `main.py`, `frontend/app.js`. Walking through real functions and variables. Open the files beside you.

---

## PART 1 — `config.py`

Pure config constants. The important ones:
```python
CLIENT_ID = os.environ.get("OUTLOOK_CLIENT_ID", "").strip()
AUTHORITY = "https://login.microsoftonline.com/common"
SCOPES = ["Mail.ReadWrite", "User.Read"]
GRAPH_BASE = "https://graph.microsoft.com/v1.0"
TOKEN_ENDPOINT = "https://login.microsoftonline.com/common/oauth2/v2.0/token"
IMAP_SCOPES = ["https://outlook.office.com/IMAP.AccessAsUser.All", "offline_access"]
IMAP_HOST, IMAP_PORT = "outlook.office365.com", 993
DEFAULT_MAIL_COUNT = 15
```
`STORAGE_DIR.mkdir(parents=True, exist_ok=True)` creates `storage/` if missing. Three paths: token cache, key, imported accounts.

---

## PART 2 — `store.py` (Fernet encryption)

- `_load_or_create_key()`: read `cache.key` or `Fernet.generate_key()` then `os.chmod(..., S_IRUSR|S_IWUSR)` (600).
- `load_cache()`: `SerializableTokenCache()`; if the file exists → `Fernet(key).decrypt(...)` → `cache.deserialize(...)`; `except Exception: pass` (skip if corrupt).
- `save_cache(cache)`: `if not cache.has_state_changed: return` → `fernet.encrypt(cache.serialize().encode())` → write → chmod 600.
- `load_imported_accounts()` / `save_imported_accounts()`: JSON ↔ Fernet, returning `{"version":1,"accounts":{}}` by default on error/empty.

---

## PART 3 — `engine.py`: INITIALIZATION & UTILITIES

```python
class OutlookEngine:
    def __init__(self):
        self._cache = load_cache()
        self._imported = load_imported_accounts()
        self._lock = threading.Lock()
        self._login_jobs = {}
        self._import_jobs = {}
```
- `_build_app()`: `msal.PublicClientApplication(CLIENT_ID, authority=AUTHORITY, token_cache=self._cache)`; raises if CLIENT_ID is missing.
- `_persist()` / `_persist_imported()`: call save_cache / save_imported_accounts.
- `_imported_id(email, client_id)`: `"imported:" + sha256(f"{email}|{client_id}")[:20]` → a stable id for an imported account.
- `_public_account(id, data)`: normalizes the info returned to the frontend (username, source, scope, can_write...).
- `_classify_graph_error` / `_classify_exception`: map errors → token_expired / missing_scope / graph_error / imap_error / mailbox_error. `_health_label`: the human-readable label.

---

## PART 4 — IMPORTING A REFRESH TOKEN

- `_parse_import_line(line)`: split on `|`, **exactly 4 parts**; assign `email_address, _password, refresh_token, client_id` (password goes into `_password` → **discarded**); validate email has `@`, has a refresh_token, has a client_id.
- `_redeem_refresh_token(client_id, refresh_token, scopes)`:
```python
resp = requests.post(TOKEN_ENDPOINT, data={
    "client_id": client_id, "grant_type": "refresh_token",
    "refresh_token": refresh_token, "scope": " ".join(scopes)})
# error / no access_token → raise RuntimeError(_token_error(data))
return data   # {access_token, refresh_token?, ...}
```
- `_graph_import_scope_options()`: a list of 6 scope sets to try (Mail.Read, Mail.ReadWrite, full-URL variants, with User.Read...), each with `can_write` + a label.
- `_try_graph_import(parsed, refresh_token)`: loop over the scope sets, redeem then `GET /me/messages?$top=1&$select=id`; **200 = usable** → return (token_data, scopes, can_write, label, new refresh_token). Exhausted → raise a combined error.
- `import_refresh_token_account(line)`:
  1. parse → try Graph (`_try_graph_import`) + fetch `displayName`.
  2. Graph fails → **fall back to IMAP**: redeem IMAP_SCOPES + `_imap_probe`.
  3. Both fail → raise a combined "Graph: ... | IMAP: ...".
  4. Save the account (mode graph/imap, scopes, can_write, new refresh_token) into `_imported` (under Lock) + persist.
- `update_imported_refresh_token(id, refresh_token)`: rebuild the import line with the new token then call `import_refresh_token_account` again.
- `_refresh_imported_access_token(account, scopes)`: redeem; if the refresh token **rotates** (new ≠ old) → save it (Lock).

---

## PART 5 — THE MULTI-THREADED IMPORT JOB

- `_import_line_result(idx, line)`: call `import_refresh_token_account`, return `{line, email, ok, source/scope/account}` or `{line, email, ok:false, error}`.
- `import_refresh_token_lines(lines)`: import **sequentially** (for 1 line / a synchronous API).
- `begin_import_refresh_token_job(lines, concurrency)`:
```python
workers = min(max(concurrency,1),5); workers = min(workers, len(lines))
work_queue = Queue(); push enumerate(lines,1) in
job = {status:"queued", total, processed:0, added:0, failed:0,
       current_lines:[], cancel_requested:False, concurrency:workers,
       active_workers:workers, results:[]}
def _worker(id):
    while True:
        if job["cancel_requested"]: break
        try: idx,line = work_queue.get_nowait()
        except Empty: break
        result = self._import_line_result(idx, line)
        with self._lock: update processed/added/failed/results (safely)
    finally: active_workers--; if 0 → status = cancelled/done
# start `workers` daemon threads running _worker
```
- `import_job_status(job_id, offset, limit)`: return a job snapshot + a **log page** (sorted by line, sliced by offset/limit, limit≤200).
- `cancel_import_job(job_id)`: set `cancel_requested=True`, status="cancelling".

---

## PART 6 — LOGIN (DEVICE CODE)

- `begin_login()`:
```python
flow = app.initiate_device_flow(scopes=SCOPES)   # {user_code, verification_uri, ...}
job_id = uuid; _login_jobs[job_id] = {status:"pending"}
def _wait():
    result = app.acquire_token_by_device_flow(flow)   # BLOCKS until the user finishes/expires
    if "access_token" in result: self._persist(); job → success + account
    else: job → error
threading.Thread(target=_wait, daemon=True).start()
return {job_id, user_code, verification_uri, message, expires_in}
```
- `login_status(job_id)`: return the job (`pending`/`success`/`error`/`unknown`).

---

## PART 7 — ACCOUNT MANAGEMENT & HEALTH

- `list_accounts()`: merge MSAL (`app.get_accounts()` → source "graph", can_update_token=False) + imported (`_public_account`).
- `remove_account(id)`: remove from `_imported` or from MSAL (`app.remove_account`).
- `_token_for(id)`: **the token entry point** — imported graph → `_refresh_imported_access_token`; imported imap → raise; MSAL → `acquire_token_silent(SCOPES, account)`.
- `check_account_health(id)`: IMAP → connect + select INBOX readonly; Graph → `GET /me/messages?$top=1`; return ok + status + detail; classify errors.
- `account_unread_counts()`: for each account call `inbox_unread_count` (Graph `unreadItemCount` / IMAP search UNSEEN); error → null.

---

## PART 8 — IMAP (the `_imap_*` block)

- `_imap_authenticate(mail, username, token)`: `AUTHENTICATE XOAUTH2` with `user=<u>\x01auth=Bearer <token>\x01\x01`.
- `_imap_connect(account)`: refresh the access token (IMAP_SCOPES) → `IMAP4_SSL` → authenticate.
- `_imap_probe(username, token)`: connect + `select INBOX readonly` (check the IMAP token is alive).
- `_imap_select`: try candidate folder names (`_imap_folder_candidates`: sentitems→["Sent","Sent Items"]...).
- `_encode_imap_id(folder, uid)` / `_decode_imap_id(message_id)`: `imap:<b64(folder)>:<uid>` (validate uid is numeric).
- `_extract_body(msg)`: prefer `text/html`, else `text/plain` (walking `msg.walk()`, skipping attachments).
- `_imap_search_uids(...)`: build IMAP criteria (ALL/UNSEEN/FROM/SINCE/BEFORE/TEXT) → `UID SEARCH`.
- `_imap_summary` / `_imap_message_detail` / `_imap_list_attachments` / `_imap_get_attachment` / `_imap_mark_read` (STORE ±FLAGS \Seen) / `_imap_search_messages`.

---

## PART 9 — READING MAIL (GRAPH, with an IMAP branch)

- `_build_filter(unread_only, has_attachments, date_from, date_to)`: assemble `$filter` (`isRead eq false and hasAttachments eq true and receivedDateTime ge ...`).
- `get_messages(id, count, skip, folder, ...)`:
  - imported imap → `_imap_get_messages`.
  - graph → build the URL `/me/messages` (or `/me/mailFolders/{folder}/messages`) `?$top&$skip&$orderby=receivedDateTime desc&$select=...` + `$filter` (quoted); GET with Bearer; map to `{id, subject, from_name/address, received, preview, is_read, has_attachments, web_link}`; `has_more = bool(@odata.nextLink)`; **client-side sender filtering**.
- `get_unified_inbox(...)`: loop `list_accounts()` (filter `only_account` if given), call `get_messages`, **wrap each account in try/except** (error → `entry["error"]`).
- `get_message_detail(id, message_id)`: graph `GET /me/messages/{id}?$select=...,body,toRecipients,ccRecipients` → return `body_type` + `body` (HTML/text) + to/cc; imap → `_imap_message_detail`.
- `search_messages(query)` / `_search_one`: graph `?$search="..."` + header `ConsistencyLevel: eventual`; imap → `_imap_search_messages`. Each account wrapped against errors.
- `list_attachments` / `get_attachment` (graph: `contentBytes` base64 → decode; imap: from the part), `mark_read` (graph PATCH isRead; blocked if `graph_can_write=False`; imap STORE), `list_folders`, `inbox_unread_count`.

---

## PART 10 — EMAIL CHECKING

- `_valid_email_address`: regex `[^@\s]+@[^@\s]+\.[^@\s]+`.
- `_resolve_mx(domain)`: `domain.encode("idna")` (IDN) → GET Cloudflare DoH `?name=<domain>&type=MX&Accept: application/dns-json` → filter answers where `type==15` (MX), take the host, sort unique; handle nxdomain/no_mx/errors.
- `_infer_email_provider(domain, mx)`: guess microsoft/google/yahoo/zoho/yandex/unknown from the string.
- `check_email_addresses(addresses)`: for each address → validate → MX (cached per domain) → status `domain_can_receive` / `syntax_error` / `nxdomain` / `no_mx`. Return `{total, ok, failed, results}`.

---

## PART 11 — `routes.py` & `main.py`

- `routes.py`: `APIRouter(prefix="/api/outlook")`; every route does `from engine import engine` (lazy) → calls a method → catches RuntimeError → HTTPException. `FOLDER_MAP` translates folder names. The attachment-download endpoint returns a `Response` with `Content-Disposition: attachment; filename*=UTF-8''<quote(name)>`.
- `main.py`: FastAPI + CORS(`*`) + include the router + `/api/health` + `mount("/", StaticFiles(html=True))` last.

---

## PART 12 — `frontend/app.js`

- `api(path, opts)`: fetch + extract the `.detail` on error.
- `state`: folder, searchMode, order/loaded/pages/hasMore (pagination per account), accounts, accountHealth, importJob.
- **Accounts**: `loadStatus`→`renderAccounts` (table of email/source/scope/updated/unread/health), `checkAccountHealth`, `openTokenModal`/`saveTokenUpdate`, `loadUnreadCounts`.
- **Import**: `startImportLines`→POST job→`pollImportJob`(1.5s)→`renderImportJob`/`renderImportProgress`/`renderImportLog`/`renderImportPager`; `loadAllImportResults` (gather pages of 200) → `failedImportLines`/`copyFailedLines`/`retryFailedLines`; `cancelImportJob`.
- **Login**: `startLogin`→show code/link→`pollLogin`(2.5s).
- **Inbox**: `loadInbox`(skip 0, build order/loaded/pages/hasMore)/`loadMore`(skip=pages×PAGE)/`renderInbox`/`mailItemHtml`; `filterParams()` gathers the query; folder tabs.
- **Search**: `doSearch`/`clearSearch`.
- **Mail detail**: `openMail` → `iframe.srcdoc = body` (HTML) / `<pre>escape` (text); auto `setRead(true, false)`; `loadAttachments` (skip inline); `setRead` toggle.
- **Email check**: `checkEmailsLive` + `renderEmailCheckResults`.

---

## 🎯 Quick "function ↔ purpose" reference (engine.py)

| Group | Main functions |
|------|----------------|
| Init | `__init__`, `_build_app`, `_persist`, `_imported_id`, `_public_account` |
| Login | `begin_login`, `login_status` |
| Token | `_token_for`, `_redeem_refresh_token`, `_refresh_imported_access_token` |
| Import | `_parse_import_line`, `_try_graph_import`, `import_refresh_token_account`, `update_imported_refresh_token`, `import_refresh_token_lines` |
| Import job | `begin_import_refresh_token_job`, `import_job_status`, `cancel_import_job`, `_import_line_result` |
| Account | `list_accounts`, `remove_account`, `check_account_health`, `account_unread_counts` |
| Reading mail (Graph) | `get_messages`, `get_unified_inbox`, `get_message_detail`, `search_messages`/`_search_one`, `mark_read`, `list_attachments`, `get_attachment`, `list_folders`, `inbox_unread_count`, `_build_filter` |
| IMAP | `_imap_authenticate/connect/probe/select`, `_encode/_decode_imap_id`, `_extract_body`, `_imap_search_uids`, `_imap_summary/_message_detail/_get_messages/_list_folders/_mark_read/_search_messages/_list_attachments/_get_attachment` |
| Email check | `check_email_addresses`, `_resolve_mx`, `_infer_email_provider`, `_valid_email_address` |
| Error classification | `_classify_graph_error`, `_classify_exception`, `_health_label` |

> 💡 Master this table + read file 08 and you can **point at any function in engine.py and explain it**.
