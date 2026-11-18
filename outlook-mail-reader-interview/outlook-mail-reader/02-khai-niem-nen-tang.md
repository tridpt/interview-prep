# 2. Các khái niệm nền tảng (lý thuyết)

File này giải thích **mọi thuật ngữ** trong dự án, cho người chưa biết gì. Đây là dự án nặng về **OAuth/identity** nên phần này rất quan trọng.

---

## 2.1. OAuth 2.0 — Ủy quyền truy cập (cốt lõi của dự án)

**Vấn đề:** làm sao một app đọc được mail của bạn mà **không cần biết mật khẩu** của bạn?

**OAuth 2.0** giải quyết: bạn đăng nhập trực tiếp với Microsoft (không qua app), Microsoft cấp cho app một **token** (vé) để truy cập tài nguyên trong phạm vi (**scope**) bạn cho phép. App không bao giờ thấy mật khẩu.

**Các khái niệm OAuth cần nhớ:**
- **Access token**: vé ngắn hạn (vài chục phút – 1 giờ) để gọi API. Hết hạn phải lấy lại.
- **Refresh token**: vé dài hạn để **đổi lấy access token mới** mà không cần đăng nhập lại. Đây là thứ app lưu (mã hóa) để "đăng nhập 1 lần, dùng mãi".
- **Scope**: phạm vi quyền. Dự án xin `Mail.ReadWrite` (đọc mail + đánh dấu đã đọc) + `User.Read`. `offline_access` là scope để **được cấp refresh token**.
- **Client ID**: định danh app đã đăng ký với Microsoft (Azure App Registration).
- **Authority**: máy chủ định danh, ở đây `login.microsoftonline.com/common` (`common` = cả tài khoản cá nhân outlook.com lẫn tài khoản tổ chức).
- **Public client**: app không có "client secret" (vì là app desktop/native chạy ở máy người dùng) → cần bật "Allow public client flows" trên Azure.

---

## 2.2. Device Code Flow — cách đăng nhập của dự án

Có nhiều "luồng" OAuth. Dự án dùng **device code flow** vì hợp với app không có trình duyệt nhúng:

1. App gọi Microsoft → nhận về một **mã ngắn (user_code)** + **một link** (`verification_uri`, thường là `microsoft.com/devicelogin`).
2. App hiển thị mã + link cho người dùng.
3. Người dùng mở link trên trình duyệt bất kỳ, nhập mã, đăng nhập Microsoft thật.
4. Trong khi đó app **chạy nền chờ (poll)** Microsoft cho tới khi người dùng xong → nhận token.

**Trong dự án:** `begin_login()` tạo flow + chạy một **thread nền** (`acquire_token_by_device_flow` chặn cho tới khi xong); frontend **poll** `/login/status/{job_id}` mỗi 2.5s để biết đã xong chưa. Đây là mẫu **long-running task + polling** điển hình.

---

## 2.3. MSAL — Microsoft Authentication Library

**MSAL là gì?** Thư viện chính chủ của Microsoft lo toàn bộ phần khó của OAuth:
- `PublicClientApplication(client_id, authority, token_cache)` — đối tượng app.
- `initiate_device_flow` / `acquire_token_by_device_flow` — device code flow.
- `acquire_token_silent(scopes, account)` — **lấy access token mới mà không hỏi người dùng** (tự dùng refresh token trong cache). Đây là "phép màu" giúp không phải đăng nhập lại.
- `get_accounts()` / `remove_account()` — quản lý **nhiều tài khoản trong cùng một cache**.
- `SerializableTokenCache` — cache có thể serialize ra chuỗi để lưu/đọc.

➡️ **Điểm hay:** MSAL quản lý refresh token cho **nhiều tài khoản** trong một cache. Dự án chỉ thêm phần **mã hóa cache** trước khi ghi đĩa.

---

## 2.4. Microsoft Graph API

**Graph API** là REST API hợp nhất của Microsoft 365 để truy cập mail, lịch, danh bạ... Base URL: `https://graph.microsoft.com/v1.0`.

Các endpoint dự án dùng (gọi bằng `requests` kèm header `Authorization: Bearer <access_token>`):
- `GET /me/messages` — danh sách mail (kèm `$top`, `$skip`, `$orderby`, `$select`, `$filter`).
- `GET /me/mailFolders/{folder}/messages` — mail theo thư mục.
- `GET /me/messages/{id}` — chi tiết 1 mail (body HTML/text).
- `GET /me/messages?$search="..."` — tìm full-text (cần header `ConsistencyLevel: eventual`).
- `PATCH /me/messages/{id}` `{isRead: true/false}` — đánh dấu đã đọc.
- `GET .../attachments` — đính kèm.
- `GET /me/mailFolders` — danh sách thư mục + số chưa đọc.

**OData query** ($top/$skip/$filter/$select/$orderby/$search): cú pháp truy vấn của Graph. Ví dụ `$filter=isRead eq false and hasAttachments eq true`.

---

## 2.5. IMAP + XOAUTH2 (chế độ dự phòng)

Một số tài khoản/refresh token chỉ dùng được cho **IMAP** (giao thức đọc mail cũ) chứ không cho Graph. Dự án có fallback:
- Kết nối `imaplib.IMAP4_SSL("outlook.office365.com", 993)`.
- Xác thực bằng **XOAUTH2**: thay vì mật khẩu, gửi chuỗi `user=<email>\x01auth=Bearer <access_token>\x01\x01` (vẫn dùng OAuth token, không phải mật khẩu).
- Đọc mail bằng lệnh IMAP (`SELECT`, `UID SEARCH`, `UID FETCH`), parse bằng `email`/`BytesParser`.

➡️ Dự án **viết song song hai cách** (Graph và IMAP) cho mọi thao tác (list/detail/search/mark-read/attachments) — chọn theo `mode` của tài khoản.

---

## 2.6. Mã hóa token bằng Fernet (cryptography)

**Vì sao mã hóa?** Refresh token là "chìa khóa" truy cập mail — nếu lưu plaintext mà file bị lộ thì nguy hiểm. Dự án mã hóa trước khi ghi đĩa.

- **Fernet** (trong thư viện `cryptography`): mã hóa đối xứng tiện dụng (AES-128-CBC + HMAC), `encrypt`/`decrypt` đơn giản.
- **Khóa mã hóa** sinh một lần (`Fernet.generate_key()`), lưu ở `storage/cache.key`, đặt quyền **chỉ chủ sở hữu đọc** (`chmod 600` trên POSIX; Windows bỏ qua an toàn).
- Token cache MSAL (`SerializableTokenCache.serialize()`) và file "imported accounts" đều được Fernet mã hóa rồi mới ghi.
- Khi đọc: giải mã; nếu hỏng/đổi khóa → bỏ qua, người dùng đăng nhập lại (không làm sập app).

> 💡 **Câu hỏi: "Mã hóa bằng khóa lưu ngay cạnh file mã hóa thì có ý nghĩa gì?"** Trả lời thật: nó **không** chống được kẻ tấn công đã chiếm toàn quyền máy (vì khóa nằm cùng máy). Nó chống **lộ ngẫu nhiên** (sao lưu nhầm, đẩy nhầm lên git, người khác đọc lướt file). Bảo vệ mạnh hơn cần khóa từ OS keychain/biến môi trường — đây là **giới hạn đã biết** nên nêu.

---

## 2.7. Đa luồng & job nền (threading, Queue)

Import nhiều refresh token (mỗi cái phải gọi mạng để kiểm chứng) sẽ chậm nếu làm tuần tự. Dự án dùng:
- **`Queue`**: hàng đợi các dòng cần xử lý (an toàn đa luồng).
- **Worker threads** (1-5, do người dùng chọn): mỗi luồng lấy việc từ queue xử lý song song.
- **`threading.Lock`**: bảo vệ trạng thái job dùng chung (đếm processed/added/failed) khỏi tranh chấp (race condition).
- Job lưu progress, log; frontend **poll** `/import-jobs/{id}` để hiện %; có nút **hủy** (đặt cờ `cancel_requested`, worker tự dừng).

➡️ Đây là mẫu **producer/consumer + job tracking** kinh điển, làm thủ công bằng thư viện chuẩn.

---

## 2.8. DNS-over-HTTPS & MX (tính năng check email)

Khi "check email live", dự án không gửi mail thử mà tra cứu **bản ghi MX (Mail eXchanger)** của domain:
- Gọi **Cloudflare DNS-over-HTTPS** (`cloudflare-dns.com/dns-query?name=<domain>&type=MX`) — tra DNS qua HTTPS (không cần thư viện DNS).
- Nếu domain có MX → **có khả năng nhận mail**; đoán provider (microsoft/google/yahoo...) từ tên MX.
- **Không xác minh mailbox cụ thể** (vì chỉ có địa chỉ thì không thể chắc chắn hộp thư tồn tại).
- Xử lý domain quốc tế (IDN) bằng `encode("idna")`.

---

## 2.9. An toàn render HTML mail — iframe sandbox

Mail HTML từ người lạ có thể chứa **mã độc/script/tracker**. Render thẳng vào trang là rủi ro XSS. Dự án dùng:
- **`<iframe sandbox>`**: nhúng HTML mail vào một iframe bị "cách ly" — `sandbox` (không có giá trị) **vô hiệu hóa toàn bộ**: không chạy script, không form, không cho thoát ra trang cha.
- Đặt nội dung qua `srcdoc`. Mail dạng text thì bọc trong `<pre>` đã `escapeHtml`.

> 💡 Đây là điểm bảo mật rất đáng nói: "Em render HTML mail trong iframe `sandbox` để cô lập hoàn toàn — script trong mail không chạy được, không tracker, không thoát ra DOM chính."

---

## 2.10. Một số thuật ngữ khác

- **FastAPI / ASGI / Uvicorn / Pydantic**: như các dự án Python khác (web async, validate request bằng model).
- **CORS**: `allow_origins=["*"]` — cho mọi origin gọi API (tiện dev; là điểm cần siết khi public — xem file 06).
- **Lazy import** `from engine import engine` **bên trong** mỗi route: để `engine` (singleton, tạo lúc import) không bị khởi tạo sớm gây lỗi vòng lặp import.
- **Polling**: client hỏi server lặp lại theo chu kỳ để biết trạng thái (đăng nhập, job import).
- **Singleton**: `engine = OutlookEngine()` ở cuối file — một instance dùng chung toàn app.
- **home_account_id**: định danh tài khoản trong MSAL; với tài khoản import là `imported:<sha256 rút gọn>`.
