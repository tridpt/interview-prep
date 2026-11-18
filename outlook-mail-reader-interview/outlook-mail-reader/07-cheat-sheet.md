# 7. Cheat Sheet — Xem nhanh trước giờ G

Đọc lướt 5-10 phút trước khi vào phỏng vấn.

---

## 🎯 Pitch 30 giây (học thuộc)
> Outlook Mail Reader gộp hộp thư **nhiều tài khoản Outlook/M365** vào một chỗ. Mỗi tài khoản **đăng nhập OAuth một lần** (device code flow), sau đó app tự dùng **refresh token** (lưu **mã hóa**) để lấy mail mà không đăng nhập lại — **không lưu mật khẩu**. Backend **FastAPI + MSAL + Microsoft Graph** (và **IMAP XOAUTH2** dự phòng): unified inbox, tìm kiếm, đánh dấu đã đọc, đính kèm, **import token đa luồng** có tiến độ/hủy, check email bằng **DNS/MX**. Frontend **JS thuần**, render HTML mail trong **iframe sandbox**.

## 🧱 Tech stack
- **Backend**: FastAPI + Uvicorn, **MSAL** (OAuth/token), **requests** (Graph + DoH), **cryptography/Fernet** (mã hóa), **imaplib** (IMAP), **threading/queue** (job).
- **Frontend**: HTML/CSS/JS thuần; **iframe sandbox** cho HTML mail.
- **Dịch vụ**: login.microsoftonline.com (OAuth), graph.microsoft.com, outlook.office365.com (IMAP), cloudflare-dns.com (MX).
- **Cấu hình**: `OUTLOOK_CLIENT_ID` (Azure App, public client).

## 🗂️ File
| File | Vai trò |
|---|---|
| config.py | client_id, scope, endpoint, đường dẫn |
| store.py | mã hóa Fernet token cache + imported accounts |
| engine.py (1524) | OutlookEngine: OAuth, token, Graph+IMAP, import job, check email |
| routes.py | API /api/outlook/* (lazy import engine) |
| main.py | FastAPI + CORS + mount frontend |
| frontend/app.js (810) | UI: login poll, inbox, search, import job, iframe sandbox |

## 🔑 OAuth (NHỚ KỸ)
- **Access token**: ngắn hạn, gọi API. **Refresh token**: dài hạn, đổi lấy access mới (lưu mã hóa).
- **Scope**: Mail.ReadWrite + User.Read + offline_access (để có refresh token).
- **Device code flow**: app cho mã + link → user nhập ở trình duyệt → app poll tới khi xong.
- **MSAL**: PublicClientApplication, `acquire_token_by_device_flow`, **`acquire_token_silent`** (tự refresh), token cache đa tài khoản.
- **Public client** + "Allow public client flows" = Yes trên Azure.

## 🔐 Lưu token mã hóa
serialize MSAL cache → **Fernet (AES) encrypt** → token_cache.bin; khóa ở cache.key (chmod 600); ghi khi `has_state_changed`. **Giới hạn**: khóa cùng máy → chống lộ ngẫu nhiên, không chống kẻ chiếm máy.

## 🔀 Hai chế độ auth (mode)
- **graph (MSAL)**: device login → silent refresh.
- **imported**: dán refresh token → `_redeem_refresh_token` + **dò scope** → Graph; không được thì **IMAP** (XOAUTH2 + probe INBOX).
- Mọi thao tác đọc mail kiểm `mode` rồi rẽ Graph/IMAP.

## 🌐 Graph + IMAP
- Graph: REST + Bearer; `/me/messages` ($top/$skip/$orderby/$select/$filter), `$search`+ConsistencyLevel, PATCH isRead, /attachments, /mailFolders.
- IMAP: `IMAP4_SSL outlook.office365.com:993`, **XOAUTH2** (`user=..\x01auth=Bearer ..`), SELECT/UID SEARCH/UID FETCH, FLAGS `\Seen`, id mail = `imap:<b64folder>:<uid>`.

## 🧵 Import job đa luồng
Queue + 1-5 worker thread + **threading.Lock** (chống race) → progress % + log phân trang + hủy (cờ cancel_requested) + copy/retry dòng lỗi. Đăng nhập cũng dùng **thread nền + poll**.

## 🛡️ Bảo mật
1. OAuth, **không mật khẩu** (password trong dòng import bị bỏ).
2. Token **mã hóa Fernet**.
3. HTML mail trong **iframe sandbox** (không chạy script); escapeHtml chuỗi động.
4. Mark-read chặn nếu token chỉ Mail.Read.

## 📨 Check email
format + **MX qua Cloudflare DoH** + đoán provider + IDN. **Không** xác minh mailbox cụ thể.

## 🧱 Kiến trúc
config ← store ← engine (singleton, Lock) ← routes (lazy import) ← main. Backend **stateful** (token file + jobs RAM). Cô lập lỗi mỗi tài khoản.

## ⚖️ Trade-off / giới hạn
Khóa mã hóa cùng máy; CORS `*` (chạy local); engine.py lớn (1524 dòng); unified inbox gọi tuần tự (chậm khi nhiều account); job RAM mất khi restart. Import token cần ràng buộc "chỉ tài khoản mình sở hữu".

## 🚀 Cải thiện tương lai
Khóa từ OS keychain; xác thực API + siết CORS; song song hóa unified inbox; pool IMAP; tách engine.py; soạn/gửi mail, thông báo mail mới, TTS.

## 💬 Thuật ngữ chốt
- **OAuth 2.0 / access token / refresh token / scope / offline_access**
- **MSAL / device code flow / acquire_token_silent / public client**
- **Microsoft Graph / OData ($filter/$search) / Bearer**
- **IMAP / XOAUTH2 / UID / FLAGS \Seen**
- **Fernet/AES / cache.key / chmod 600**
- **threading / Queue / Lock / daemon / polling**
- **iframe sandbox / escapeHtml / XSS**
- **DNS-over-HTTPS / MX record / IDN**

---

### ✅ Checklist trước phỏng vấn
- [ ] Pitch dự án 30 giây.
- [ ] Access vs refresh token + device code flow.
- [ ] MSAL & vì sao không tự viết OAuth.
- [ ] Lưu token mã hóa + giới hạn (khóa cùng máy).
- [ ] Vì sao có cả Graph và IMAP (adapter theo mode).
- [ ] Import job đa luồng (Queue/Lock/cancel).
- [ ] iframe sandbox render HTML an toàn.
- [ ] 2 hạn chế + cách khắc phục.
- [ ] 2 câu hỏi ngược nhà tuyển dụng.

**Hít thở sâu. Bạn hiểu dự án này. Tự tin kể câu chuyện của mình. 🍀**
