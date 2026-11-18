# 📚 Bộ tài liệu ôn phỏng vấn — Dự án OTP Vault (totp-manager)

Chào bạn. Bộ tài liệu này giúp bạn **đi từ con số 0 đến tự tin trả lời phỏng vấn** về dự án OTP Vault — trình quản lý mã 2FA (TOTP). Mỗi file đi sâu một khía cạnh. Đọc theo thứ tự dưới đây.

> Toàn bộ tài liệu được viết sau khi **đọc trực tiếp 100% mã nguồn thật**: backend (`config.py`, `totp.py`, `store.py`, `main.py`), bộ test (RFC 6238 vectors + property-based hypothesis + FastAPI TestClient), và frontend (`app.js`, `index.html`, `sw.js`). Không suy đoán.

---

## 🗺️ Lộ trình học (đọc theo thứ tự)

| # | File | Nội dung | Đọc khi |
|---|------|----------|---------|
| 1 | `01-tong-quan-du-an.md` | Dự án là gì, giải quyết vấn đề gì, tech stack, cách chạy | Đầu tiên |
| 2 | `02-khai-niem-nen-tang.md` | Lý thuyết nền: TOTP/RFC 6238, HMAC, base32, scrypt, Fernet/AES, FastAPI, PWA, property-based test | Khi chưa rõ thuật ngữ |
| 3 | `03-kien-truc-he-thong.md` | Kiến trúc, luồng dữ liệu, mô hình bảo mật, vòng đời phiên | Sau khi nắm khái niệm |
| 4 | `04-backend-chi-tiet.md` | Mổ xẻ từng file backend, từng hàm | Học sâu phần server |
| 5 | `05-frontend-chi-tiet.md` | Mổ xẻ app.js, index.html, sw.js | Học sâu phần client |
| 6 | `06-tinh-nang-noi-bat.md` | Phân tích sâu tính năng "ăn điểm" | Để kể chuyện ấn tượng |
| 7 | `07-cau-hoi-phong-van.md` | 70+ câu hỏi & câu trả lời mẫu | Luyện tập trả lời |
| 8 | `08-cheat-sheet.md` | Tóm tắt 1 trang xem trước giờ G | Phút chót |

---

## ⏱️ Nếu bạn chỉ có ít thời gian

- **Còn 1 ngày**: file 1 → 2 → 3 → 7 → 8.
- **Còn 2 giờ**: file 1 → 3 → 8, rồi lướt 7.
- **Còn 30 phút**: file 8 + phần "Câu hỏi hay gặp nhất" trong file 7.

---

## 📦 Tóm tắt dự án trong 3 câu (học thuộc)

> OTP Vault là một web app **quản lý mã 2FA (TOTP)** chạy cục bộ — nhập secret key, nhận mã 6 số đổi mỗi 30 giây, cùng chuẩn Google Authenticator/Authy (**RFC 6238**). Toàn bộ tài khoản được lưu trong một **kho mã hóa** (`vault.bin`) bảo vệ bằng **mật khẩu chủ**: khóa mã hóa được suy ra từ mật khẩu qua **scrypt** và **không bao giờ lưu ra đĩa** — quên mật khẩu là mất dữ liệu (đúng theo thiết kế). Backend là **FastAPI (Python)** dùng `hmac`/`hashlib` chuẩn để sinh TOTP và **Fernet (AES) + scrypt** để mã hóa kho; frontend là **HTML/CSS/JS thuần (PWA)** có quét QR (jsQR), kéo-thả sắp xếp, sao lưu/khôi phục. Dự án có **bộ test kỹ lưỡng** gồm test vector RFC 6238 và **property-based testing (hypothesis)**.

Chúc bạn phỏng vấn thật tốt! 🍀
