# 1. Tổng quan dự án Outlook Mail Reader

## 1.1. Outlook Mail Reader là gì?

Outlook Mail Reader là một **web app đọc hộp thư** gộp mail của **nhiều tài khoản Outlook / Microsoft 365** vào một giao diện duy nhất (unified inbox). Ý tưởng cốt lõi: mỗi tài khoản chỉ phải **đăng nhập Microsoft đúng một lần**; sau đó app dùng **refresh token** (lưu mã hóa) để tự lấy mail các lần sau mà **không hỏi đăng nhập lại** và **không bao giờ lưu mật khẩu**.

### Điểm cốt lõi cần nhớ
- **Đa tài khoản**: gộp inbox của nhiều account vào một chỗ.
- **OAuth 2.0 chuẩn Microsoft** (Microsoft Graph) — đăng nhập 1 lần qua màn hình Microsoft thật.
- **Không lưu mật khẩu**; refresh token lưu **mã hóa** (Fernet/AES) dưới đĩa.
- **Hai chế độ xác thực**: (1) đăng nhập tương tác (device code flow) qua MSAL; (2) import refresh token có sẵn (Graph hoặc IMAP).
- Backend **FastAPI**, frontend **JS thuần**; mail HTML render trong **iframe sandbox**.

## 1.2. Dự án giải quyết vấn đề gì?

1. **Phải mở nhiều tab/đăng nhập nhiều tài khoản Outlook** → gộp tất cả vào một inbox.
2. **Đăng nhập lại phiền phức** → đăng nhập 1 lần, sau đó tự refresh token.
3. **Bảo mật** → dùng OAuth chuẩn, không lưu mật khẩu, token mã hóa.
4. **Quản lý lô tài khoản** → import refresh token hàng loạt, job đa luồng có tiến độ/hủy, kiểm tra "sống/chết" của token, kiểm tra email theo DNS/MX.
5. **Trình diễn năng lực kỹ thuật**: tích hợp OAuth/Graph thực tế, mã hóa, đa luồng, fallback IMAP, an toàn render HTML — phần "ăn điểm" khi phỏng vấn.

> 💡 **Mẹo phỏng vấn:** đây là dự án **tích hợp OAuth/identity thực tế** — thứ rất giá trị vì hầu hết sản phẩm doanh nghiệp đều cần "đăng nhập bằng Microsoft/Google". Hãy nhấn vào việc bạn hiểu **luồng OAuth 2.0, refresh token, và lưu trữ token an toàn** — đó là kiến thức cốt lõi của backend hiện đại.

## 1.3. Các tính năng chính

| Nhóm | Tính năng |
|------|-----------|
| **Đăng nhập** | Device code flow (mở link, nhập mã), chạy nền + poll trạng thái |
| **Unified inbox** | Gộp mail nhiều tài khoản; phân trang, lọc (chưa đọc/đính kèm/người gửi/ngày/tài khoản), theo thư mục (inbox/sent/drafts/junk/deleted/archive) |
| **Xem mail** | Nội dung đầy đủ, HTML render trong iframe sandbox, danh sách + tải đính kèm |
| **Đánh dấu** | Đã đọc / chưa đọc (cần Mail.ReadWrite); badge số chưa đọc mỗi tài khoản |
| **Tìm kiếm** | Full-text trên tất cả tài khoản cùng lúc (Graph `$search` / IMAP TEXT) |
| **Quản lý account** | Xem nguồn (Graph/IMAP), scope, thời gian cập nhật token; đổi token; xóa; kiểm tra "sống" |
| **Import token** | Dán nhiều dòng `email|password|refresh_token|client_id`; bỏ qua password, chỉ lưu token mã hóa |
| **Import job** | Chạy nền 1-5 luồng, progress %, log phân trang, hủy job, copy/retry dòng lỗi |
| **Check email** | Kiểm tra format + DNS/MX + đoán provider (không xác minh mailbox cụ thể) |

## 1.4. Công nghệ sử dụng (tech stack)

**Backend (Python):**
- **FastAPI** `0.115.0` — web framework, REST API.
- **Uvicorn** `[standard] 0.30.6` — ASGI server.
- **MSAL** `>=1.31.0` (Microsoft Authentication Library) — OAuth/đăng nhập + quản lý token cache đa tài khoản.
- **requests** `>=2.32.0` — gọi Microsoft Graph API và DNS-over-HTTPS.
- **cryptography** `>=43.0.0` — **Fernet** (AES) mã hóa token cache dưới đĩa.
- **imaplib** (thư viện chuẩn) — fallback IMAP XOAUTH2.
- **threading / queue** (chuẩn) — job import đa luồng.

**Frontend:**
- **HTML + CSS + JavaScript thuần** (không framework, không build).
- **iframe `sandbox`** để render HTML mail an toàn; `escapeHtml` cho nội dung động.

**Dịch vụ ngoài:**
- **Microsoft identity platform** (`login.microsoftonline.com`) — OAuth.
- **Microsoft Graph** (`graph.microsoft.com/v1.0`) — đọc mail.
- **Outlook IMAP** (`outlook.office365.com:993`) — fallback.
- **Cloudflare DNS-over-HTTPS** (`cloudflare-dns.com/dns-query`) — tra cứu MX khi check email.

**Cấu hình:** `OUTLOOK_CLIENT_ID` (Azure App Registration, public client) qua biến môi trường.

> 💡 **Câu hỏi: "Vì sao dùng MSAL thay vì tự viết OAuth?"** → MSAL là thư viện chính chủ của Microsoft, lo việc khó: device code flow, **tự refresh token**, quản lý **token cache đa tài khoản**, xử lý hết hạn. Tự viết OAuth dễ sai và kém an toàn. Mình chỉ cần bọc thêm phần mã hóa cache khi ghi đĩa.

## 1.5. Cách chạy

```bash
cd outlook-mail-reader
python -m venv .venv
.venv\Scripts\activate                     # Windows
set OUTLOOK_CLIENT_ID=<client-id-cua-ban>  # client id từ Azure App Registration
pip install -r requirements.txt
cd backend
uvicorn main:app --reload --host 0.0.0.0 --port 8809
```
Mở `http://localhost:8809`. Trên Windows có `start_outlook_reader.bat` tự tạo venv + cài + set client_id + chạy.

**Thiết lập Azure (1 lần):** portal.azure.com → App registrations → New registration (Supported account types = *any org directory + personal Microsoft accounts*) → Authentication → bật **Allow public client flows** = Yes → copy **Application (client) ID**.

## 1.6. Cấu trúc thư mục

```
outlook-mail-reader/
├── backend/
│   ├── config.py    # client_id, scope, đường dẫn, endpoint
│   ├── store.py     # lưu token cache đã mã hóa (Fernet/AES)
│   ├── engine.py    # đăng nhập device-code, tự refresh, gọi Graph/IMAP (1524 dòng)
│   ├── routes.py    # API /api/outlook/*
│   └── main.py      # FastAPI app + phục vụ frontend
├── frontend/        # index.html, style.css, app.js
└── storage/         # token_cache.bin, imported_accounts.bin, cache.key (đều mã hóa/bí mật)
```

> ⚠️ **Lưu ý đạo đức & pháp lý (dự án tự nêu):** Dùng OAuth chuẩn của Microsoft, **không lưu mật khẩu**, và mỗi tài khoản vẫn phải qua màn đăng nhập Microsoft thật một lần — nên **chỉ dùng được với tài khoản bạn sở hữu hoặc có quyền truy cập**. Quyền yêu cầu là `Mail.ReadWrite` (đọc + đánh dấu đã đọc, **không gửi/xóa**). Khi phỏng vấn, nên trình bày dự án như một **mail client cá nhân đa tài khoản** và nhấn vào khía cạnh kỹ thuật OAuth/bảo mật.
