# 3. Kiến trúc hệ thống

## 3.1. Sơ đồ tổng thể

```
┌───────────────────────── TRÌNH DUYỆT (frontend) ─────────────────────────┐
│  index.html + style.css + app.js (JS thuần)                               │
│  - device login (poll), unified inbox, search, filters, import job UI     │
│  - render HTML mail trong <iframe sandbox>                                │
└───────────────────────────────┬───────────────────────────────────────────┘
                                │ HTTP (fetch) /api/outlook/*
                                ▼
┌──────────────────────── BACKEND — FastAPI (main.py) ──────────────────────┐
│  routes.py  — định nghĩa endpoint, validate (Pydantic), lazy import engine │
│       │                                                                    │
│       ▼                                                                    │
│  engine.py — OutlookEngine (singleton): đăng nhập, token, Graph/IMAP,      │
│              import job đa luồng, check email, sanitize                    │
│       │                  │                         │                       │
│       ▼                  ▼                         ▼                       │
│  store.py (mã hóa)   MSAL (token cache)      requests / imaplib            │
│  Fernet/AES          PublicClientApp                                       │
└───────┬──────────────────┬─────────────────────────┬──────────────────────┘
        │                  │                          │
        ▼                  ▼                          ▼
  storage/*.bin     login.microsoftonline.com   graph.microsoft.com
  (token mã hóa)    (OAuth)                      outlook.office365.com (IMAP)
                                                 cloudflare-dns.com (MX)
```

**Đặc điểm cốt lõi:** đây là **backend tích hợp identity/OAuth thực tế**, không chỉ CRUD. `engine.py` là trái tim (1524 dòng) — nơi xử lý token, gọi Graph/IMAP, đa luồng. Backend **có lưu trạng thái** (token cache, login jobs, import jobs trong RAM + file mã hóa) — khác các app tĩnh client-only.

## 3.2. Phân lớp & trách nhiệm các file

| File | Vai trò |
|------|---------|
| `config.py` | Hằng cấu hình: `CLIENT_ID`, `AUTHORITY`, `SCOPES`, endpoint Graph/IMAP/token, đường dẫn storage |
| `store.py` | Mã hóa Fernet: load/save token cache MSAL + imported accounts; quản lý khóa |
| `engine.py` | `OutlookEngine`: đăng nhập, refresh token, Graph + IMAP, import job đa luồng, check email, sanitize. **Singleton `engine`** |
| `routes.py` | `APIRouter` `/api/outlook/*`: nhận request, validate, gọi engine, map lỗi → HTTP |
| `main.py` | Tạo FastAPI app, CORS, include router, `/api/health`, mount frontend tĩnh |
| `frontend/*` | UI: gọi API, render inbox/mail, quản lý job import, iframe sandbox |

## 3.3. Hai chế độ xác thực (quan trọng nhất)

Đây là điểm kiến trúc đặc sắc — mỗi tài khoản thuộc một trong các `mode`:

### Mode "graph" qua MSAL (đăng nhập tương tác)
- Người dùng đăng nhập bằng **device code flow**.
- MSAL lưu refresh token vào **token cache dùng chung** (mã hóa qua `store.py`).
- Lấy token sau này: `acquire_token_silent()` (MSAL tự refresh).
- `home_account_id` do MSAL cấp.

### Mode "imported" (dán refresh token có sẵn)
- Người dùng dán dòng `email|password|refresh_token|client_id` (password **bị bỏ qua**).
- Engine **tự đổi refresh token** lấy access token (`_redeem_refresh_token` → POST `TOKEN_ENDPOINT`), **thử nhiều bộ scope** (`_graph_import_scope_options`: Mail.Read/Mail.ReadWrite...) để dò quyền.
- Nếu Graph không dùng được → thử **IMAP** (mode "imap"): redeem scope IMAP rồi `_imap_probe`.
- Lưu trong `imported_accounts.bin` (mã hóa riêng), `home_account_id = imported:<sha256>`.

➡️ **Mọi thao tác đọc mail** (`get_messages`, `get_message_detail`, `search`, `mark_read`, attachments, folders, unread count) đều **kiểm tra mode** rồi rẽ nhánh Graph hoặc IMAP. Đây là một **lớp trừu tượng hai backend** dưới cùng một API.

## 3.4. Luồng đăng nhập (device code) — học kỹ

```
1. [Frontend] bấm "+ Thêm tài khoản" → POST /login/start
2. [Engine] begin_login(): MSAL initiate_device_flow → {user_code, verification_uri}
   + tạo job_id + chạy THREAD nền: acquire_token_by_device_flow (chặn tới khi xong)
3. [Frontend] hiện mã + link; poll GET /login/status/{job_id} mỗi 2.5s
4. [User] mở link, nhập mã, đăng nhập Microsoft thật
5. [Thread nền] nhận token → save_cache() (mã hóa) → job.status = "success"
6. [Frontend] thấy "success" → ẩn khu đăng nhập, reload danh sách tài khoản
```

## 3.5. Luồng đọc unified inbox

```
[Frontend] GET /inbox?folder=&count=&skip=&unread_only=&sender=&date_from=...&account=
   │
[routes] map folder thân thiện → well-known folder của Graph (FOLDER_MAP)
   ▼
[engine.get_unified_inbox] lặp từng tài khoản:
   - graph → get_messages() dựng URL Graph ($top/$skip/$orderby/$select/$filter)
   - imap  → _imap_get_messages() (SELECT + UID SEARCH + UID FETCH + parse)
   - lỗi 1 tài khoản → ghi "error" cho tài khoản đó, KHÔNG làm hỏng cả response
   ▼
trả [{account, home_account_id, messages, has_more, error?}]
```
Mỗi tài khoản được bọc try/except riêng → **một tài khoản chết không kéo sập cả inbox**.

## 3.6. Import job đa luồng

```
POST /accounts/import-refresh-token/job {line: "...nhiều dòng...", concurrency: 1-5}
   ▼
begin_import_refresh_token_job:
   - đẩy mọi dòng vào Queue
   - khởi N worker thread (daemon); mỗi worker: lấy dòng → import_refresh_token_account → ghi kết quả
   - Lock bảo vệ job state (processed/added/failed/current_lines)
   - worker cuối cùng đặt status = done/cancelled
   ▼
[Frontend] poll GET /import-jobs/{id}?offset=&limit= → progress % + log phân trang
           có nút Hủy (POST .../cancel → cờ cancel_requested)
```

## 3.7. State của backend nằm ở đâu?

- **Lâu dài (file mã hóa)**: `token_cache.bin` (MSAL cache), `imported_accounts.bin`, `cache.key`.
- **Tạm thời (RAM)**: `_login_jobs`, `_import_jobs` (dict trong engine singleton) — mất khi restart server.
- ➡️ Backend **stateful** (khác các app client-only). Một instance `engine` dùng chung, có `threading.Lock` cho an toàn đa luồng.

## 3.8. Vì sao kiến trúc này hợp lý?

- **Tách lớp rõ**: config ← store (mã hóa) ← engine (logic+mạng) ← routes (HTTP) ← frontend.
- **Trừu tượng hai backend** (Graph/IMAP) sau một API thống nhất → frontend không cần biết tài khoản dùng cơ chế nào.
- **Cô lập lỗi**: mỗi tài khoản/dòng import bọc try/except → lỗi cục bộ không lan.
- **Bảo mật theo lớp**: OAuth (không mật khẩu) + token mã hóa + iframe sandbox cho HTML mail.
