# 8. Cheat Sheet — Xem nhanh trước giờ G

Đọc lướt 5-10 phút trước khi vào phỏng vấn.

---

## 🎯 Pitch 30 giây (học thuộc)
> OTP Vault: web app quản lý mã **2FA (TOTP)** chạy cục bộ — nhập secret, ra mã 6 số đổi mỗi 30s theo **RFC 6238**. Nhiều tài khoản lưu trong **kho mã hóa** bảo vệ bằng **mật khẩu chủ**: khóa suy ra từ mật khẩu qua **scrypt**, mã hóa bằng **Fernet (AES)**, **khóa chỉ ở RAM**, tự khóa sau 5 phút. Backend **FastAPI** tự implement TOTP từ thư viện chuẩn; frontend **vanilla JS (PWA)** có quét QR (jsQR). Test gồm **test vector RFC 6238 + property-based (hypothesis)**.

## 🧱 Tech stack
- **Backend**: FastAPI + uvicorn; `cryptography` (Scrypt + Fernet); `segno` (QR SVG); TOTP tự viết bằng `hmac/hashlib/base64/struct`.
- **Frontend**: vanilla HTML/CSS/JS; jsQR (vendor); PWA (sw.js).
- **Test**: pytest + hypothesis + httpx (TestClient).

## 🗂️ File
| File | Vai trò |
|---|---|
| config.py | đường dẫn vault, tham số scrypt, auto-lock 300s |
| totp.py | TOTP (HMAC+truncation), normalize_secret, parse/build otpauth |
| store.py | Session: setup/unlock/lock, CRUD, export/import, scrypt+Fernet |
| main.py | FastAPI /api/*, phục vụ frontend |
| frontend/app.js | gọi API, render, đếm ngược, quét QR, kéo-thả, theme |
| frontend/sw.js | PWA network-first, KHÔNG cache /api |

## 🔢 TOTP (nhớ từng bước)
base32 decode secret → counter = floor(time/30) → `struct.pack(">Q", counter)` → HMAC(key, counter, sha1) → **dynamic truncation** (offset = digest[-1]&0xF, đọc 4 byte, `&0x7FFFFFFF`) → `% 10^digits` → pad 0. SHA1/256/512. `seconds_remaining` ∈ [1, period].

## 🔐 Mã hóa (mô hình cốt lõi)
mật khẩu → **scrypt(salt, N=32768)** → khóa 32 byte → **Fernet** (AES-128-CBC + HMAC-SHA256). `vault.bin = {salt, ciphertext}`. **Khóa CHỈ ở RAM** (Session.fernet). **unlock = thử decrypt** (InvalidToken → sai mật khẩu). Quên mật khẩu = mất dữ liệu (no backdoor). File không chứa secret rõ (có test).

## 🔁 Session & auto-lock
`is_unlocked()` = có fernet + chưa quá 300s + file còn tồn tại. Thao tác → `touch()`. Quá hạn / file xóa → `lock()` → API 401 → frontend hiện màn khóa.

## 🌐 API (cần mở khóa hay không)
KHÔNG cần: status, setup, unlock, lock, **generate** (sinh mã từ secret, không đụng kho).
CẦN: accounts (GET/POST/PATCH/DELETE), reorder, import-uri, reveal (QR+secret), export, import, export-csv, import-csv.
Status: 400 (request sai), 401 (khóa/sai mật khẩu), 404 (không tìm thấy).

## 🖼️ Frontend
`api()` bọc fetch (ném Error theo detail). `setInterval(loadAccounts,1000)` làm mới mã + đếm ngược. Quét QR bằng **jsQR ở client** (ảnh không rời máy, chỉ gửi otpauth). Kéo-thả → reorder. escapeHtml chống XSS. SW **không cache /api**.

## 🧪 Test (ĐIỂM NHẤN)
- **Test vector RFC 6238** (SHA1/256/512 × 4 mốc) → đúng tuyệt đối.
- **Property-based (hypothesis)**: seconds_remaining ∈ [1,period]; parse(build(...)) round-trip.
- **3 tầng**: totp (thuật toán) / store (kho) / api (TestClient).
- **conftest**: vault tạm + giảm scrypt + reset session → không đụng dữ liệu thật.
- **test_vault_encrypted_at_rest**: file thô không lộ secret.
- Spec-driven: test gắn "Req X.Y".

## 🛡️ Bảo mật (nhớ điểm)
Khóa không lưu đĩa · local-only 127.0.0.1 · auto-lock · os.chmod(0o600) · CSV cảnh báo secret rõ · SW không cache /api · escapeHtml.

## ⚖️ Trade-off
Không lưu khóa → bảo mật cao, quên mật khẩu = mất dữ liệu. Local-only → an toàn, không truy cập từ xa. Tự viết TOTP → ít phụ thuộc, phải tự đảm bảo đúng (bù bằng test vector).

## 🚀 Cải thiện tương lai
Rate-limit thử mật khẩu; tăng scrypt N theo phần cứng; mã hóa từng tài khoản; cửa sổ time skew khi xác thực; multi-user nếu triển khai chung.

## 💬 Thuật ngữ chốt
- **TOTP / HOTP / RFC 6238 / RFC 4226 / HMAC / base32 / dynamic truncation**
- **scrypt (KDF) / Fernet (AES) / salt / mã hóa-tại-chỗ / zero-knowledge**
- **Session / auto-lock / master password**
- **FastAPI / Pydantic / StaticFiles / 401**
- **jsQR / getUserMedia / Canvas / PWA / network-first**
- **pytest / hypothesis / property-based / RFC test vectors / spec-driven**

---

### ✅ Checklist trước phỏng vấn
- [ ] Pitch 30 giây.
- [ ] Giải thích TOTP từng bước + vì sao tự viết.
- [ ] Mô hình mã hóa "không lưu khóa" + unlock xác thực.
- [ ] Auto-lock & xử lý ca biên.
- [ ] Test vector RFC + property-based.
- [ ] Quét QR client & SW không cache /api.
- [ ] 2 hạn chế + cách khắc phục.
- [ ] 2 câu hỏi ngược nhà tuyển dụng.

**Hít thở sâu. Bạn hiểu dự án này. Tự tin kể câu chuyện của mình. 🍀**
