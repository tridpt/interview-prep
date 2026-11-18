# 8. Phụ lục: Đi bộ qua code (Code Walkthrough)

> ⚠️ Viết sau khi **đọc trực tiếp 100% mã nguồn**: `config.py`, `store.py`, `engine.py` (1524 dòng — đọc toàn bộ), `routes.py`, `main.py`, `frontend/app.js`. Đi qua hàm thật, biến thật. Mở file bên cạnh để đối chiếu.

---

## PHẦN 1 — `config.py`

Hằng cấu hình thuần. Quan trọng:
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
`STORAGE_DIR.mkdir(parents=True, exist_ok=True)` tạo `storage/` nếu chưa có. Ba đường dẫn: token cache, key, imported accounts.

---

## PHẦN 2 — `store.py` (mã hóa Fernet)

- `_load_or_create_key()`: đọc `cache.key` hoặc `Fernet.generate_key()` rồi `os.chmod(..., S_IRUSR|S_IWUSR)` (600).
- `load_cache()`: `SerializableTokenCache()`; nếu file tồn tại → `Fernet(key).decrypt(...)` → `cache.deserialize(...)`; `except Exception: pass` (hỏng thì bỏ qua).
- `save_cache(cache)`: `if not cache.has_state_changed: return` → `fernet.encrypt(cache.serialize().encode())` → ghi → chmod 600.
- `load_imported_accounts()` / `save_imported_accounts()`: JSON ↔ Fernet, trả `{"version":1,"accounts":{}}` mặc định khi lỗi/trống.

---

## PHẦN 3 — `engine.py`: KHỞI TẠO & TIỆN ÍCH

```python
class OutlookEngine:
    def __init__(self):
        self._cache = load_cache()
        self._imported = load_imported_accounts()
        self._lock = threading.Lock()
        self._login_jobs = {}
        self._import_jobs = {}
```
- `_build_app()`: `msal.PublicClientApplication(CLIENT_ID, authority=AUTHORITY, token_cache=self._cache)`; raise nếu thiếu CLIENT_ID.
- `_persist()` / `_persist_imported()`: gọi save_cache / save_imported_accounts.
- `_imported_id(email, client_id)`: `"imported:" + sha256(f"{email}|{client_id}")[:20]` → id ổn định cho tài khoản import.
- `_public_account(id, data)`: chuẩn hóa thông tin trả ra frontend (username, source, scope, can_write...).
- `_classify_graph_error` / `_classify_exception`: ánh xạ lỗi → token_expired / missing_scope / graph_error / imap_error / mailbox_error. `_health_label`: nhãn tiếng Việt.

---

## PHẦN 4 — IMPORT REFRESH TOKEN

- `_parse_import_line(line)`: split `|`, **đúng 4 phần**; gán `email_address, _password, refresh_token, client_id` (password vào `_password` → **bỏ**); validate email có `@`, có refresh_token, có client_id.
- `_redeem_refresh_token(client_id, refresh_token, scopes)`:
```python
resp = requests.post(TOKEN_ENDPOINT, data={
    "client_id": client_id, "grant_type": "refresh_token",
    "refresh_token": refresh_token, "scope": " ".join(scopes)})
# lỗi/không có access_token → raise RuntimeError(_token_error(data))
return data   # {access_token, refresh_token?, ...}
```
- `_graph_import_scope_options()`: danh sách 6 bộ scope thử (Mail.Read, Mail.ReadWrite, bản full URL, kèm User.Read...), mỗi bộ kèm `can_write` + label.
- `_try_graph_import(parsed, refresh_token)`: lặp các bộ scope, redeem rồi `GET /me/messages?$top=1&$select=id`; **200 = dùng được** → trả (token_data, scopes, can_write, label, refresh_token mới). Hết → raise gộp lỗi.
- `import_refresh_token_account(line)`:
  1. parse → thử Graph (`_try_graph_import`) + lấy `displayName`.
  2. Graph fail → **fallback IMAP**: redeem IMAP_SCOPES + `_imap_probe`.
  3. Cả hai fail → raise gộp "Graph: ... | IMAP: ...".
  4. Lưu account (mode graph/imap, scopes, can_write, refresh_token mới) vào `_imported` (có Lock) + persist.
- `update_imported_refresh_token(id, refresh_token)`: dựng lại dòng import với token mới rồi gọi lại `import_refresh_token_account`.
- `_refresh_imported_access_token(account, scopes)`: redeem; nếu refresh token **xoay vòng** (mới ≠ cũ) → lưu lại (Lock).

---

## PHẦN 5 — IMPORT JOB ĐA LUỒNG

- `_import_line_result(idx, line)`: gọi `import_refresh_token_account`, trả `{line, email, ok, source/scope/account}` hoặc `{line, email, ok:false, error}`.
- `import_refresh_token_lines(lines)`: import **tuần tự** (dùng cho 1 dòng / API đồng bộ).
- `begin_import_refresh_token_job(lines, concurrency)`:
```python
workers = min(max(concurrency,1),5); workers = min(workers, len(lines))
work_queue = Queue(); đẩy enumerate(lines,1) vào
job = {status:"queued", total, processed:0, added:0, failed:0,
       current_lines:[], cancel_requested:False, concurrency:workers,
       active_workers:workers, results:[]}
def _worker(id):
    while True:
        if job["cancel_requested"]: break
        try: idx,line = work_queue.get_nowait()
        except Empty: break
        result = self._import_line_result(idx, line)
        with self._lock: cập nhật processed/added/failed/results (an toàn)
    finally: active_workers--; nếu 0 → status = cancelled/done
# khởi `workers` daemon thread chạy _worker
```
- `import_job_status(job_id, offset, limit)`: trả snapshot job + **trang log** (sort theo line, cắt offset/limit, limit≤200).
- `cancel_import_job(job_id)`: đặt `cancel_requested=True`, status="cancelling".

---

## PHẦN 6 — ĐĂNG NHẬP (DEVICE CODE)

- `begin_login()`:
```python
flow = app.initiate_device_flow(scopes=SCOPES)   # {user_code, verification_uri, ...}
job_id = uuid; _login_jobs[job_id] = {status:"pending"}
def _wait():
    result = app.acquire_token_by_device_flow(flow)   # CHẶN tới khi user xong/hết hạn
    if "access_token" in result: self._persist(); job → success + account
    else: job → error
threading.Thread(target=_wait, daemon=True).start()
return {job_id, user_code, verification_uri, message, expires_in}
```
- `login_status(job_id)`: trả job (`pending`/`success`/`error`/`unknown`).

---

## PHẦN 7 — QUẢN LÝ TÀI KHOẢN & HEALTH

- `list_accounts()`: gộp MSAL (`app.get_accounts()` → source "graph", can_update_token=False) + imported (`_public_account`).
- `remove_account(id)`: xóa khỏi `_imported` hoặc khỏi MSAL (`app.remove_account`).
- `_token_for(id)`: **điểm vào token** — imported graph → `_refresh_imported_access_token`; imported imap → raise; MSAL → `acquire_token_silent(SCOPES, account)`.
- `check_account_health(id)`: IMAP → connect + select INBOX readonly; Graph → `GET /me/messages?$top=1`; trả ok + status + detail; phân loại lỗi.
- `account_unread_counts()`: với mỗi account gọi `inbox_unread_count` (Graph `unreadItemCount` / IMAP search UNSEEN); lỗi → null.

---

## PHẦN 8 — IMAP (khối `_imap_*`)

- `_imap_authenticate(mail, username, token)`: `AUTHENTICATE XOAUTH2` với `user=<u>\x01auth=Bearer <token>\x01\x01`.
- `_imap_connect(account)`: refresh access token (IMAP_SCOPES) → `IMAP4_SSL` → authenticate.
- `_imap_probe(username, token)`: connect + `select INBOX readonly` (kiểm token IMAP sống).
- `_imap_select`: thử các tên thư mục ứng viên (`_imap_folder_candidates`: sentitems→["Sent","Sent Items"]...).
- `_encode_imap_id(folder, uid)` / `_decode_imap_id(message_id)`: `imap:<b64(folder)>:<uid>` (validate uid là số).
- `_extract_body(msg)`: ưu tiên `text/html`, không thì `text/plain` (đi qua `msg.walk()`, bỏ attachment).
- `_imap_search_uids(...)`: dựng criteria IMAP (ALL/UNSEEN/FROM/SINCE/BEFORE/TEXT) → `UID SEARCH`.
- `_imap_summary` / `_imap_message_detail` / `_imap_list_attachments` / `_imap_get_attachment` / `_imap_mark_read` (STORE ±FLAGS \Seen) / `_imap_search_messages`.

---

## PHẦN 9 — ĐỌC MAIL (GRAPH, có rẽ nhánh IMAP)

- `_build_filter(unread_only, has_attachments, date_from, date_to)`: ghép `$filter` (`isRead eq false and hasAttachments eq true and receivedDateTime ge ...`).
- `get_messages(id, count, skip, folder, ...)`:
  - imported imap → `_imap_get_messages`.
  - graph → dựng URL `/me/messages` (hoặc `/me/mailFolders/{folder}/messages`) `?$top&$skip&$orderby=receivedDateTime desc&$select=...` + `$filter` (quote); GET với Bearer; map sang `{id, subject, from_name/address, received, preview, is_read, has_attachments, web_link}`; `has_more = bool(@odata.nextLink)`; **lọc sender phía client**.
- `get_unified_inbox(...)`: lặp `list_accounts()` (lọc `only_account` nếu có), gọi `get_messages`, **bọc try/except từng tài khoản** (lỗi → `entry["error"]`).
- `get_message_detail(id, message_id)`: graph `GET /me/messages/{id}?$select=...,body,toRecipients,ccRecipients` → trả `body_type` + `body` (HTML/text) + to/cc; imap → `_imap_message_detail`.
- `search_messages(query)` / `_search_one`: graph `?$search="..."` + header `ConsistencyLevel: eventual`; imap → `_imap_search_messages`. Bọc lỗi từng tài khoản.
- `list_attachments` / `get_attachment` (graph: `contentBytes` base64 → decode; imap: lấy từ part), `mark_read` (graph PATCH isRead; chặn nếu `graph_can_write=False`; imap STORE), `list_folders`, `inbox_unread_count`.

---

## PHẦN 10 — CHECK EMAIL

- `_valid_email_address`: regex `[^@\s]+@[^@\s]+\.[^@\s]+`.
- `_resolve_mx(domain)`: `domain.encode("idna")` (IDN) → GET Cloudflare DoH `?name=<domain>&type=MX&Accept: application/dns-json` → lọc answer `type==15` (MX), lấy host, sort unique; xử lý nxdomain/no_mx/lỗi.
- `_infer_email_provider(domain, mx)`: đoán microsoft/google/yahoo/zoho/yandex/unknown từ chuỗi.
- `check_email_addresses(addresses)`: với mỗi địa chỉ → validate → MX (cache theo domain) → status `domain_can_receive` / `syntax_error` / `nxdomain` / `no_mx`. Trả `{total, ok, failed, results}`.

---

## PHẦN 11 — `routes.py` & `main.py`

- `routes.py`: `APIRouter(prefix="/api/outlook")`; mọi route `from engine import engine` (lazy) → gọi method → bắt RuntimeError → HTTPException. `FOLDER_MAP` đổi tên thư mục. Endpoint tải đính kèm trả `Response` với `Content-Disposition: attachment; filename*=UTF-8''<quote(name)>`.
- `main.py`: FastAPI + CORS(`*`) + include router + `/api/health` + `mount("/", StaticFiles(html=True))` cuối cùng.

---

## PHẦN 12 — `frontend/app.js`

- `api(path, opts)`: fetch + lấy `.detail` lỗi.
- `state`: folder, searchMode, order/loaded/pages/hasMore (phân trang theo account), accounts, accountHealth, importJob.
- **Account**: `loadStatus`→`renderAccounts` (bảng email/nguồn/scope/cập nhật/chưa đọc/health), `checkAccountHealth`, `openTokenModal`/`saveTokenUpdate`, `loadUnreadCounts`.
- **Import**: `startImportLines`→POST job→`pollImportJob`(1.5s)→`renderImportJob`/`renderImportProgress`/`renderImportLog`/`renderImportPager`; `loadAllImportResults` (gom phân trang 200) → `failedImportLines`/`copyFailedLines`/`retryFailedLines`; `cancelImportJob`.
- **Login**: `startLogin`→hiện mã/link→`pollLogin`(2.5s).
- **Inbox**: `loadInbox`(skip 0, dựng order/loaded/pages/hasMore)/`loadMore`(skip=pages×PAGE)/`renderInbox`/`mailItemHtml`; `filterParams()` gom query; tab thư mục.
- **Search**: `doSearch`/`clearSearch`.
- **Mail detail**: `openMail` → `iframe.srcdoc = body` (HTML) / `<pre>escape` (text); tự `setRead(true, false)`; `loadAttachments` (bỏ inline); `setRead` toggle.
- **Check email**: `checkEmailsLive` + `renderEmailCheckResults`.

---

## 🎯 Bảng tra cứu nhanh "hàm ↔ chức năng" (engine.py)

| Nhóm | Hàm chính |
|------|-----------|
| Khởi tạo | `__init__`, `_build_app`, `_persist`, `_imported_id`, `_public_account` |
| Đăng nhập | `begin_login`, `login_status` |
| Token | `_token_for`, `_redeem_refresh_token`, `_refresh_imported_access_token` |
| Import | `_parse_import_line`, `_try_graph_import`, `import_refresh_token_account`, `update_imported_refresh_token`, `import_refresh_token_lines` |
| Import job | `begin_import_refresh_token_job`, `import_job_status`, `cancel_import_job`, `_import_line_result` |
| Account | `list_accounts`, `remove_account`, `check_account_health`, `account_unread_counts` |
| Đọc mail Graph | `get_messages`, `get_unified_inbox`, `get_message_detail`, `search_messages`/`_search_one`, `mark_read`, `list_attachments`, `get_attachment`, `list_folders`, `inbox_unread_count`, `_build_filter` |
| IMAP | `_imap_authenticate/connect/probe/select`, `_encode/_decode_imap_id`, `_extract_body`, `_imap_search_uids`, `_imap_summary/_message_detail/_get_messages/_list_folders/_mark_read/_search_messages/_list_attachments/_get_attachment` |
| Check email | `check_email_addresses`, `_resolve_mx`, `_infer_email_provider`, `_valid_email_address` |
| Phân loại lỗi | `_classify_graph_error`, `_classify_exception`, `_health_label` |

> 💡 Nắm bảng này + đọc file 08 là bạn có thể **chỉ tay vào bất kỳ hàm nào trong engine.py và giải thích**.
