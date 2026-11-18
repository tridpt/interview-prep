# 6. Phân tích sâu các tính năng "ăn điểm"

Mỗi tính năng kèm: ý tưởng, cách làm, "vì sao hay".

---

## 6.1. Tự cài đặt TOTP đúng RFC 6238 ⭐⭐ (đặc sắc nhất)

**Ý tưởng:** sinh mã 2FA cùng chuẩn Google Authenticator **mà không dùng thư viện TOTP** — chỉ `hmac`/`hashlib`/`base64`/`struct` chuẩn.

**Cách làm:** decode secret base32 → HMAC(key, counter=time//30) → **dynamic truncation (RFC 4226)** → mã 6 số. Hỗ trợ SHA1/256/512, digits/period tùy biến.

**Vì sao hay:**
- Thể hiện **hiểu thuật toán mật mã** ở mức cài đặt, không chỉ gọi API.
- Kiểm chứng bằng **test vector chính thức của RFC 6238** → đúng tuyệt đối.
- Giảm phụ thuộc = giảm bề mặt tấn công.

**Có thể bị hỏi:** *"Vì sao dùng `at` parameter?"* → để test với thời điểm cố định (khớp bảng RFC). *"Truncation để làm gì?"* → biến digest 20 byte thành số ngắn dễ gõ, phân bố đều.

---

## 6.2. Mô hình mã hóa "không lưu khóa" ⭐⭐

**Ý tưởng:** khóa giải mã **không bao giờ ghi ra đĩa**; chỉ tồn tại trong RAM khi đã mở khóa.

**Cách làm:** mật khẩu chủ → **scrypt(salt)** → khóa 32 byte → **Fernet (AES+HMAC)** mã hóa danh sách. File `vault.bin` chỉ có `{salt, ciphertext}`. `unlock` = thử giải mã (sai mật khẩu → `InvalidToken`).

**Vì sao hay:**
- Đây là mô hình **zero-knowledge với file**: lấy được file cũng vô dụng nếu không có mật khẩu.
- Dùng đúng công cụ: **scrypt** (KDF chống brute-force) + **Fernet** (mã hóa xác thực) — không tự chế mật mã.
- Có test `test_vault_encrypted_at_rest` chứng minh file thô không lộ secret.

**Câu hỏi vàng:** *"Quên mật khẩu thì sao?"* → Mất dữ liệu, **đúng theo thiết kế** (không có cửa hậu). Đây là đánh đổi bảo mật có chủ đích.

---

## 6.3. Auto-lock & quản lý phiên an toàn ⭐

**Ý tưởng:** kho tự khóa sau 5 phút không thao tác để giảm rủi ro để máy mở khóa quên.

**Cách làm:** `Session.is_unlocked()` kiểm `last_activity`; mỗi thao tác `touch()`. Khi quá hạn hoặc **file vault bị xóa** → tự `lock()`. Client phát hiện 401 → hiện lại màn khóa.

**Vì sao hay:** xử lý cả ca biên (file bị xóa giữa phiên → không 500), và frontend phản ứng đúng. Có test cho cả hai.

---

## 6.4. Bộ test nghiêm túc: RFC vectors + property-based ⭐⭐

**Ý tưởng:** với code thuật toán/bảo mật, test phải **chắc chắn đúng**.

**Cách làm:**
- **Test vector RFC 6238**: bảng cặp (thời điểm→mã) chính thức cho SHA1/256/512.
- **Property-based (hypothesis)**: `seconds_remaining` luôn ∈ [1, period] với 200 input ngẫu nhiên; `parse(build(...))` round-trip bảo toàn mọi tham số.
- **Cô lập tuyệt đối**: `conftest.py` chuyển vault sang thư mục tạm, giảm scrypt cho nhanh, reset phiên mỗi test.
- **3 tầng test**: thuật toán (test_totp), kho/mã hóa (test_store), API end-to-end (test_api qua TestClient).

**Vì sao hay:** rất "kỹ sư". Nêu được "em test bằng chuẩn chính thức + fuzzing có cấu trúc + cô lập dữ liệu thật" là điểm cộng cực lớn cho dự án bảo mật.

---

## 6.5. Quét QR ngay trong trình duyệt (jsQR) ⭐

**Ý tưởng:** thêm tài khoản bằng cách quét QR (camera hoặc ảnh) — tiện như app điện thoại.

**Cách làm:** vẽ camera/ảnh lên `canvas` → `getImageData` → **jsQR** giải mã → lấy chuỗi `otpauth://` → gửi server lưu. **Ảnh không bao giờ rời trình duyệt**, chỉ gửi chuỗi đã giải.

**Vì sao hay:** kết hợp getUserMedia + Canvas + jsQR; và quyết định **giải mã ở client** là một điểm bảo mật/riêng tư.

---

## 6.6. "Sinh mã nhanh" không cần mở khóa ⭐

**Ý tưởng:** thử ngay một secret bất kỳ mà không phải lưu hay mở kho.

**Cách làm:** `POST /generate {secret}` → trả mã + remaining; endpoint này **không gác `_ensure_unlocked`** và **không đụng kho**.

**Vì sao hay:** thể hiện tư duy tách bạch quyền truy cập — thao tác không nhạy cảm (tính toán thuần) thì không cần mở khóa.

---

## 6.7. Sao lưu/khôi phục mã hóa tự chứa + CSV có cảnh báo

- **Backup mã hóa**: gói tự chứa `salt` riêng + mật khẩu riêng → khôi phục trên máy khác; **bỏ qua trùng** (name+secret), **cấp id mới**.
- **CSV**: tiện cho di chuyển nhưng **cảnh báo rõ** secret ở dạng văn bản rõ (cả ở UI lẫn README) — thể hiện ý thức đánh đổi tiện/bảo mật.

---

## 6.8. PWA an toàn (network-first, không cache /api)

Service worker cài app được như app riêng nhưng **không cache `/api/*`** (dữ liệu nhạy cảm + mã đổi liên tục) — một quyết định bảo mật có chủ đích.

---

## 6.9. Bảng "tính năng ↔ khái niệm kỹ thuật" (học thuộc)

| Tính năng | Khái niệm bạn thể hiện |
|-----------|------------------------|
| TOTP tự viết | RFC 6238/4226, HMAC, base32, dynamic truncation |
| Mã hóa không lưu khóa | KDF (scrypt), Fernet/AES, salt, mã hóa-tại-chỗ, zero-knowledge |
| Auto-lock/Session | quản lý phiên, ca biên (file bị xóa), 401 |
| Test RFC + property-based | test vector chuẩn, hypothesis, cô lập fixture |
| Quét QR client | getUserMedia, Canvas, jsQR, riêng tư |
| generate không mở khóa | tách quyền truy cập theo độ nhạy cảm |
| Backup/CSV | round-trip, gói tự chứa, đánh đổi tiện/bảo mật |
| PWA không cache /api | service worker, bảo mật cache |

> 💡 **Chiến lược kể chuyện:** nếu được hỏi "tự hào nhất?", chọn **tự cài TOTP đúng RFC + kiểm bằng test vector** hoặc **mô hình mã hóa không lưu khóa**. Cả hai cho thấy chiều sâu bảo mật/thuật toán hiếm có.
