# 4. Code chi tiết (giải thích từng file)

Đọc kèm code thật trong `outlook-mail-reader/`. Phần đi sâu từng hàm `engine.py` xem `08-code-walkthrough.md`.

---

## 4.1. `config.py` — Cấu hình

- `BASE_DIR` / `STORAGE_DIR` (tự tạo) và đường dẫn 3 file: `TOKEN_CACHE_PATH`, `ENCRYPTION_KEY_PATH` (`cache.key`), `IMPORTED_ACCOUNTS_PATH`.
- `CLIENT_ID = os.environ.get("OUTLOOK_CLIENT_ID", "")` — lấy từ biến môi trường.
- `AUTHORITY = ".../common"` (cá nhân + tổ chức).
- `SCOPES = ["Mail.ReadWrite", "User.Read"]` (đọc + đánh dấu đã đọc, **không gửi**).
- `GRAPH_BASE`, `TOKEN_ENDPOINT`, `IMAP_SCOPES` (`IMAP.AccessAsUser.All` + `offline_access`), `IMAP_HOST/PORT` (`outlook.office365.com:993`).
- `DEFAULT_MAIL_COUNT = 15`.

> 💡 Comment trong file ghi rõ triết lý: **không lưu mật khẩu**, mỗi tài khoản đăng nhập 1 lần, sau đó MSAL tự dùng refresh token.

---

## 4.2. `store.py` — Lưu token mã hóa

**Nhiệm vụ:** serialize MSAL cache (chứa refresh token nhiều tài khoản) rồi **mã hóa Fernet** trước khi ghi đĩa.

- `_load_or_create_key()`: đọc `cache.key`; chưa có thì `Fernet.generate_key()` + `chmod 600` (POSIX; Windows bỏ qua an toàn).
- `load_cache()`: tạo `SerializableTokenCache`, nếu có file thì **giải mã** rồi `deserialize`; lỗi → bỏ qua (đăng nhập lại).
- `save_cache()`: **chỉ ghi nếu `cache.has_state_changed`** (tối ưu) → `serialize()` → `fernet.encrypt` → ghi.
- `load_imported_accounts()` / `save_imported_accounts()`: tương tự cho file imported accounts (JSON → mã hóa).

> 💡 Mọi thao tác đọc bọc try/except trả mặc định an toàn → cache hỏng không làm sập app.

---

## 4.3. `engine.py` — Trái tim (1524 dòng)

`OutlookEngine` (singleton `engine` ở cuối file). Khởi tạo: nạp cache + imported accounts, `threading.Lock`, dict `_login_jobs` / `_import_jobs`. Các nhóm (chi tiết ở file 08):

### Đăng nhập & token
- `_build_app()`: tạo `msal.PublicClientApplication(CLIENT_ID, authority, token_cache)`; lỗi nếu thiếu CLIENT_ID.
- `begin_login()` / `login_status()`: device code flow + thread nền + job tracking.
- `_token_for(home_account_id)`: lấy access token — imported graph → `_refresh_imported_access_token`; MSAL → `acquire_token_silent`. Đây là **điểm vào token cho mọi lời gọi Graph**.

### Import refresh token
- `_parse_import_line`: tách `email|password|refresh_token|client_id` (đúng 4 phần, validate email/token/client_id), **bỏ password**.
- `_redeem_refresh_token`: POST `TOKEN_ENDPOINT` đổi refresh token → access token.
- `_graph_import_scope_options` + `_try_graph_import`: **dò scope** (thử Mail.Read/Mail.ReadWrite... cho tới khi `GET /me/messages?$top=1` trả 200) để biết quyền đọc/ghi.
- `import_refresh_token_account`: thử Graph trước, không được thì **fallback IMAP** (`_imap_probe`); lưu account mã hóa.
- `import_refresh_token_lines` (tuần tự) và `begin_import_refresh_token_job` (**đa luồng**: Queue + N worker + Lock + progress/cancel) + `import_job_status` (phân trang log) + `cancel_import_job`.

### Quản lý account & health
- `list_accounts`: gộp tài khoản MSAL (`app.get_accounts()`) + imported.
- `remove_account`: xóa khỏi imported hoặc khỏi MSAL.
- `check_account_health`: IMAP mở thử INBOX hoặc Graph đọc thử 1 mail; phân loại lỗi (`_classify_*`: token_expired/missing_scope/...).
- `account_unread_counts`: số chưa đọc inbox mỗi tài khoản (cho badge).

### Đọc mail (Graph + IMAP song song)
- `get_messages` / `get_unified_inbox` / `get_message_detail` / `search_messages` / `mark_read` / `list_attachments` / `get_attachment` / `list_folders` — mỗi hàm **kiểm tra mode** rồi rẽ Graph hoặc IMAP.
- `_build_filter`: ghép `$filter` OData (isRead/hasAttachments/ngày). Lọc người gửi làm **phía client** (Graph không tìm gần đúng địa chỉ).
- Khối `_imap_*`: kết nối XOAUTH2, SELECT/SEARCH/FETCH, parse `email`/`BytesParser`, `_extract_body` (ưu tiên HTML), encode/decode id mail IMAP (`imap:<folder_b64>:<uid>`).

### Check email
- `check_email_addresses` + `_resolve_mx` (Cloudflare DoH) + `_infer_email_provider` + `_valid_email_address`.

---

## 4.4. `routes.py` — API (~210 dòng)

- `APIRouter(prefix="/api/outlook")`. Pydantic models: `ImportRefreshTokenRequest`, `UpdateRefreshTokenRequest`, `CheckEmailsRequest`.
- `FOLDER_MAP`: tên thân thiện (inbox/sent/...) → well-known folder Graph (sentitems/junkemail/...).
- Mọi route **lazy import** `from engine import engine` rồi gọi method tương ứng; bắt `RuntimeError` → `HTTPException` (400/404).
- Endpoint tải đính kèm trả `Response` với `Content-Disposition: attachment; filename*=UTF-8''<quote>` (hỗ trợ tên file Unicode).

> 💡 **Vì sao lazy import engine trong từng route?** Tránh khởi tạo `OutlookEngine()` (đọc file, dựng MSAL) quá sớm khi import module, và tránh import vòng. Engine là singleton nên import nhiều lần vẫn dùng chung một instance.

---

## 4.5. `main.py` — App

- `FastAPI(...)` + `CORSMiddleware(allow_origins=["*"], ...)`.
- `include_router(outlook_router)`.
- `/api/health` → `{status, configured: bool(CLIENT_ID)}`.
- `app.mount("/", StaticFiles(directory=frontend, html=True))` — phục vụ frontend (đặt **cuối** để không che route `/api`).

---

## 4.6. `frontend/index.html` + `app.js` (~810 dòng)

- **index.html**: các card (tài khoản, check mail, hộp thư), tab thư mục, bộ lọc, 2 modal (đổi token, chi tiết mail). Mail body là **`<iframe id="m-body" sandbox>`**.
- **app.js** (JS thuần):
  - `api(path, opts)`: bọc `fetch`, lấy `detail` lỗi từ JSON.
  - `state`: folder, searchMode, order/loaded/pages/hasMore (phân trang theo tài khoản), accounts, importJob.
  - **Account**: `loadStatus`, `renderAccounts` (bảng), `checkAccountHealth`, `openTokenModal`/`saveTokenUpdate`, `loadUnreadCounts` (badge).
  - **Import**: `startImportLines` → tạo job → `pollImportJob` (1.5s) → render progress/log/pager; `copyFailedLines`/`retryFailedLines`/`cancelImportJob`.
  - **Login**: `startLogin` → hiện mã/link → `pollLogin` (2.5s).
  - **Inbox**: `loadInbox`/`loadMore`/`renderInbox`/`mailItemHtml`; lọc qua `filterParams()`; tab thư mục.
  - **Search**: `doSearch`/`clearSearch`.
  - **Mail detail**: `openMail` đặt `iframe.srcdoc = body` (HTML) hoặc `<pre>` escape (text); tự `setRead(true)` khi mở; `loadAttachments`; toggle đã đọc.
  - **Check email**: `checkEmailsLive` + bảng kết quả.

> 💡 **Điểm bảo mật frontend:** body HTML đặt vào `iframe[sandbox].srcdoc` → script trong mail **không chạy**. Các chuỗi động khác đều qua `escapeHtml`.

---

## 4.7. Tổng kết các quyết định kỹ thuật

| Quyết định | Lý do |
|-----------|-------|
| OAuth 2.0 + MSAL, không lưu mật khẩu | Chuẩn, an toàn, MSAL tự refresh + cache đa tài khoản |
| Device code flow | Hợp app không có web redirect; UX nhập mã đơn giản |
| Mã hóa token bằng Fernet | Không để refresh token plaintext dưới đĩa |
| Trừu tượng Graph + IMAP | Hỗ trợ cả token Graph lẫn token chỉ-IMAP sau một API |
| Dò scope khi import | Tự biết token có quyền Mail.Read hay ReadWrite |
| Import job đa luồng (Queue+Lock) | Tăng tốc import lô lớn, có progress/cancel |
| Cô lập lỗi mỗi tài khoản | Một account chết không kéo sập unified inbox |
| iframe sandbox cho HTML mail | Chống XSS/script/tracker trong mail |
| Lazy import engine trong route | Tránh import vòng + khởi tạo sớm |
| Check email bằng DNS/MX (DoH) | Kiểm tra domain nhận mail mà không gửi thử |
