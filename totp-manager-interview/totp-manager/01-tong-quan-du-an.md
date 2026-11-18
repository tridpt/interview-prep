# 1. Tổng quan dự án OTP Vault (totp-manager)

## 1.1. OTP Vault là gì?

OTP Vault là một **web app quản lý mã 2FA** (two-factor authentication) chạy cục bộ trên máy bạn. Bạn nhập **secret key** của một tài khoản (lấy khi bật 2FA), app sinh ra **mã 6 số đổi mỗi 30 giây** — y hệt Google Authenticator hay Authy. Khác biệt: app lưu được **nhiều tài khoản**, hiển thị mã cho tất cả cùng lúc, và bảo vệ bằng **mật khẩu chủ + mã hóa cục bộ**.

### Điểm cốt lõi cần nhớ
- Sinh mã theo chuẩn **TOTP (RFC 6238)** — chuẩn dùng chung của mọi app xác thực.
- **Mật khẩu chủ**: khóa mã hóa suy ra từ mật khẩu qua **scrypt**, **không lưu khóa ra đĩa**. Quên mật khẩu = mất dữ liệu.
- **Tự khóa sau 5 phút** không thao tác + nút khóa thủ công + ẩn/hiện mã.
- Chạy **local-only** (`127.0.0.1`), không đa người dùng, không expose ra Internet.
- Backend **FastAPI (Python)**, frontend **vanilla JS (PWA)**, có **bộ test kỹ** (RFC vectors + property-based).

> ⚠️ Tài liệu README nêu rõ: dùng cho tài khoản **của bạn** hoặc được phép quản lý; không phải công cụ nuôi tài khoản ảo hàng loạt.

## 1.2. Dự án giải quyết vấn đề gì?

1. **Quản lý nhiều mã 2FA ở một nơi an toàn trên máy tính** — thay vì chỉ có trên điện thoại.
2. **Bảo mật dữ liệu nhạy cảm**: secret key 2FA là cực kỳ nhạy cảm; app mã hóa toàn bộ bằng mật khẩu chủ, không gửi đi đâu.
3. **Di chuyển/sao lưu**: xuất kho mã hóa, hiện lại QR/secret để thêm sang điện thoại, nhập/xuất CSV.
4. **Trình diễn năng lực kỹ thuật**: tự cài đặt thuật toán mật mã (TOTP từ HMAC), mã hóa-tại-chỗ với KDF, quản lý phiên/auto-lock, và **kiểm thử nghiêm túc** (test vector chuẩn + property-based) — đây là phần "ăn điểm" nhất.

> 💡 **Mẹo phỏng vấn:** đây là dự án **bảo mật + thuật toán mật mã** — rất khác các app CRUD thường gặp. Hãy nhấn vào: tự implement TOTP đúng RFC từ thư viện chuẩn, mô hình mã hóa "không lưu khóa", và bộ test có **test vector RFC 6238** + **property-based testing**. Những thứ này cho thấy bạn hiểu sâu, không chỉ ghép API.

## 1.3. Các tính năng chính

| Nhóm | Tính năng |
|------|-----------|
| **Sinh mã** | Sinh mã nhanh từ secret (không cần mở khóa, không lưu); hiển thị mã + đếm ngược cho mọi tài khoản |
| **Bảo mật** | Mật khẩu chủ (scrypt), mã hóa kho (Fernet/AES), tự khóa 5 phút, khóa thủ công, ẩn/hiện mã |
| **Quản lý tài khoản** | Thêm (tên + secret / dán `otpauth://` / **quét QR** camera hoặc ảnh), đổi tên, ghi chú, kéo-thả sắp xếp, xóa |
| **Hiện lại** | Hiện QR (SVG) + secret của từng tài khoản để thêm sang app khác |
| **Sao lưu** | Xuất gói **mã hóa** (mật khẩu riêng), khôi phục (bỏ qua trùng); xuất/nhập **CSV** (cảnh báo secret dạng rõ) |
| **Giao diện** | Sáng/tối (lưu localStorage), tìm kiếm, bấm mã để copy, **PWA** cài được |

## 1.4. Công nghệ sử dụng (tech stack)

**Backend (Python):**
- **FastAPI** `0.115.0` — web framework, định nghĩa REST API.
- **uvicorn[standard]** `0.30.6` — ASGI server.
- **cryptography** `>=43` — `Fernet` (mã hóa đối xứng) + `Scrypt` (KDF). **Đây là chỗ làm mã hóa.**
- **segno** `>=1.6` — sinh QR code (SVG) thuần Python cho tính năng "hiện lại QR".
- **TOTP tự viết** bằng `hmac`, `hashlib`, `base64`, `struct` của thư viện chuẩn — **không cần thư viện ngoài**.

**Frontend:**
- **HTML/CSS/JS thuần (vanilla)** — không framework, không build.
- **jsQR** (vendor, để trong repo) — giải mã QR ngay trong trình duyệt (camera/ảnh).
- Web API: Fetch, getUserMedia (camera), Canvas, Clipboard, localStorage, Service Worker (PWA).

**Test:**
- **pytest** + **hypothesis** (property-based testing) + **httpx** (cho `fastapi.testclient.TestClient`).

> 💡 **Câu hỏi cài bẫy: "Sao tự viết TOTP mà không dùng thư viện `pyotp`?"** Trả lời tốt: "TOTP chỉ là HMAC + dynamic truncation theo RFC 6238 — tự viết bằng `hmac`/`hashlib` chuẩn giúp **giảm phụ thuộc** (ít bề mặt tấn công), **hiểu rõ thuật toán**, và em đã kiểm chứng bằng **test vector chính thức của RFC 6238** (SHA1/256/512). Cái cần thư viện thật sự là phần mật mã khó (mã hóa kho) thì em dùng `cryptography` chuẩn, không tự chế."

## 1.5. Cách chạy

```bat
cd /d d:\AI_App\totp-manager
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt

cd backend
uvicorn main:app --reload --port 8200
```
Mở `http://127.0.0.1:8200` — lần đầu sẽ yêu cầu **tạo mật khẩu chủ**.

Chạy test:
```bat
pip install -r requirements-dev.txt
pytest
```

> 🔒 App **chỉ lắng nghe `127.0.0.1`** → chỉ máy này truy cập được. Dữ liệu mã hóa trong `storage/vault.bin` và `storage/` đã `.gitignore`.

## 1.6. Cấu trúc thư mục

```
totp-manager/
├── backend/
│   ├── config.py   # Đường dẫn vault, tham số scrypt, thời gian auto-lock
│   ├── totp.py     # Sinh mã TOTP (RFC 6238), normalize secret, parse/build otpauth
│   ├── store.py    # Kho mã hóa (Session): setup/unlock/lock, CRUD, export/import
│   └── main.py     # FastAPI: các endpoint /api/*, phục vụ frontend tĩnh
├── frontend/
│   ├── index.html  # Giao diện: màn khóa, sinh mã nhanh, danh sách, sao lưu, modal QR
│   ├── app.js      # Toàn bộ logic client (gọi API, render, quét QR, kéo-thả, theme)
│   ├── style.css   # Giao diện sáng/tối
│   ├── sw.js       # Service worker (PWA, network-first, KHÔNG cache /api)
│   ├── manifest.json
│   └── vendor/jsQR.js  # Thư viện giải mã QR (client)
├── tests/          # test_totp.py, test_store.py, test_api.py
├── conftest.py     # Fixture cô lập vault tạm + giảm scrypt cho test nhanh
├── storage/vault.bin   # Kho mã hóa (đã .gitignore)
├── requirements.txt / requirements-dev.txt
└── .kiro/specs/    # Đặc tả yêu cầu (spec-driven: test gắn "Req X.Y")
```

> 💡 **Điểm đáng chú ý:** dự án có thư mục `.kiro/specs/` — phát triển theo **spec-driven** (đặc tả yêu cầu trước). Các test chú thích rõ "Req 2.1", "Req 6.3"... nghĩa là mỗi test ánh xạ tới một yêu cầu cụ thể. Nêu được điều này cho thấy quy trình làm việc bài bản.
