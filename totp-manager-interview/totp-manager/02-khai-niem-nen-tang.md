# 2. Các khái niệm nền tảng (lý thuyết)

File này giải thích **mọi thuật ngữ** trong dự án, cho người chưa biết gì. Đây là dự án **bảo mật/mật mã** nên phần khái niệm rất quan trọng — nắm chắc sẽ ghi điểm lớn.

---

## 2.1. 2FA, OTP, TOTP, HOTP

- **2FA (Two-Factor Authentication)**: xác thực hai lớp — ngoài mật khẩu còn cần "yếu tố thứ hai" (mã đổi liên tục) để đăng nhập.
- **OTP (One-Time Password)**: mật khẩu dùng một lần.
- **HOTP (HMAC-based OTP, RFC 4226)**: OTP dựa trên một **bộ đếm (counter)** tăng dần.
- **TOTP (Time-based OTP, RFC 6238)**: biến thể của HOTP nhưng **counter = thời gian**. Cụ thể `counter = floor(thời_gian_hiện_tại / period)`, với `period` thường là 30 giây. Vì cả server và app đều biết thời gian, chúng tính ra **cùng một mã** mà không cần liên lạc.

➡️ Dự án này sinh **TOTP** — đúng chuẩn Google Authenticator/Authy.

---

## 2.2. Thuật toán TOTP hoạt động thế nào (cốt lõi của dự án)

Đây là phần **phải hiểu vanh vách**. Các bước trong `totp.py` → `generate_code`:

1. **Decode secret**: secret là chuỗi **base32** (chỉ chữ A-Z và số 2-7). Giải mã về bytes → đây là **khóa HMAC**.
2. **Tính counter**: `counter = int(thời_gian // period)` (số chu kỳ 30 giây kể từ mốc 1970).
3. **Đóng gói counter**: `struct.pack(">Q", counter)` → 8 byte big-endian.
4. **HMAC**: `hmac.new(key, counter_bytes, sha1).digest()` → một digest 20 byte (SHA1).
5. **Dynamic truncation (RFC 4226)**: lấy 4 bit cuối của digest làm `offset`, đọc 4 byte từ vị trí `offset`, bỏ bit dấu (`& 0x7FFFFFFF`) → một số 31-bit.
6. **Lấy mã**: `số % 10^digits` rồi pad 0 cho đủ `digits` (mặc định 6) → ra "492039".

**Vì sao bước truncation tồn tại?** Để biến digest dài thành một số ngắn, dễ gõ, mà vẫn phân bố đều và khó đoán.

> 💡 Nếu được hỏi "TOTP khác mật khẩu thường thế nào", nói: nó **không lưu/truyền mã**, hai bên cùng tính từ secret + thời gian; mã chỉ sống ~30 giây nên lộ ra cũng nhanh hết hạn.

---

## 2.3. HMAC & hàm băm (SHA1/256/512)

- **Hàm băm (hash)**: biến dữ liệu bất kỳ thành một chuỗi cố định (digest), một chiều. SHA1 → 20 byte, SHA256 → 32 byte, SHA512 → 64 byte.
- **HMAC (Hash-based Message Authentication Code)**: cách dùng hàm băm **kèm một khóa bí mật** để tạo mã xác thực. TOTP dùng HMAC(secret, counter).
- Dự án hỗ trợ 3 thuật toán qua dict `_ALGOS = {SHA1, SHA256, SHA512}`; mặc định SHA1 (chuẩn phổ biến nhất của Authenticator).

---

## 2.4. Base32 & chuẩn hóa secret

- **Base32**: cách mã hóa nhị phân bằng 32 ký tự (A-Z, 2-7). 2FA dùng base32 vì dễ gõ (không nhầm 0/O, 1/l).
- Người dùng hay copy secret có **dấu cách/gạch** (vd `JBSW Y3DP EHPK 3PXP`). Hàm `normalize_secret` xử lý: bỏ khoảng trắng/gạch, **viết hoa**, **thêm `=` padding** cho đủ bội số 8 (yêu cầu của base32), rồi thử `b32decode` để xác thực; sai → ném `TOTPError`.
- **Idempotent**: chuẩn hóa hai lần ra cùng kết quả (có test kiểm chứng) — quan trọng để không làm hỏng secret đã chuẩn.

---

## 2.5. otpauth:// URI & QR code

- Khi bật 2FA, dịch vụ thường cho một **QR code** chứa chuỗi `otpauth://totp/Issuer:account?secret=ABC&issuer=...&digits=6&period=30&algorithm=SHA1`.
- `parse_otpauth` tách chuỗi này thành tham số (name/secret/digits/period/algorithm); chỉ chấp nhận loại `totp` (từ chối `hotp`).
- `build_otpauth` làm ngược lại — dựng URI để **tạo lại QR** (thêm sang app khác). `segno` biến URI thành ảnh QR SVG.
- Frontend dùng **jsQR** để **đọc** QR từ camera/ảnh ngay trong trình duyệt (không gửi ảnh lên server).

---

## 2.6. Mã hóa kho: scrypt (KDF) + Fernet (AES)

Đây là **mô hình bảo mật cốt lõi** — phải hiểu kỹ.

- **KDF (Key Derivation Function)**: hàm biến **mật khẩu** (con người nhớ được) thành **khóa mã hóa** (bytes ngẫu nhiên mạnh). Dự án dùng **scrypt** — một KDF **chống brute-force** vì tốn nhiều **bộ nhớ + CPU** (tham số `N=32768, r=8, p=1`, ra khóa 32 byte).
- **Salt**: 16 byte ngẫu nhiên lưu kèm vault; cùng mật khẩu nhưng salt khác → khóa khác → chống bảng tra cứu (rainbow table).
- **Fernet**: một "công thức" mã hóa đối xứng an toàn của thư viện `cryptography` — bên trong là **AES-128-CBC + HMAC-SHA256** (vừa mã hóa **vừa xác thực toàn vẹn**). Khóa Fernet = `base64(scrypt_output_32byte)`.
- **Mô hình "không lưu khóa"**: khóa Fernet chỉ tồn tại trong **bộ nhớ server** khi đã mở khóa; file `vault.bin` chỉ chứa `{salt, data}` (data đã mã hóa). **Quên mật khẩu = mất dữ liệu** — đúng theo thiết kế (zero-knowledge với chính file).

> 💡 Câu hỏi vàng: "Nếu ai đó lấy được file vault.bin thì sao?" → Họ chỉ có salt + ciphertext; không có mật khẩu thì không dẫn được khóa (scrypt rất chậm để brute-force), không giải mã được.

---

## 2.7. Phiên & auto-lock (Session)

- Khi mở khóa, server giữ một đối tượng `Session` chứa `fernet` (khóa) + `salt` + `last_activity` **trong RAM**.
- `is_unlocked()` trả True chỉ khi: có `fernet`, **chưa quá `AUTO_LOCK_SECONDS` (300s)** kể từ `last_activity`, và file vault còn tồn tại.
- Mỗi thao tác đọc/ghi gọi `touch()` để dời mốc auto-lock. `lock()` xóa khóa khỏi RAM.
- Đây là một **biến module toàn cục** `session` — vì app chạy **cục bộ, một người dùng** (không cần multi-session/đăng nhập).

---

## 2.8. FastAPI & REST

- **FastAPI**: web framework Python bất đồng bộ, tự validate request bằng **Pydantic** (các class `...Request(BaseModel)`).
- **REST API**: client gọi các endpoint `/api/*` qua HTTP (GET/POST/PATCH/DELETE), trả JSON.
- **Status code dùng trong dự án**: `200` OK, `400` request sai (mật khẩu ngắn, secret sai, CSV thiếu cột), `401` kho đang khóa / sai mật khẩu, `404` không tìm thấy tài khoản.
- **StaticFiles**: FastAPI phục vụ luôn frontend tĩnh (mount `/` ở cuối để không che route `/api`).

---

## 2.9. PWA & Service Worker

- **PWA**: cài web app như app riêng (cửa sổ riêng, icon) qua `manifest.json` + service worker.
- **Service worker** ở đây: chiến lược **network-first** (luôn ưu tiên bản mới từ server, cache làm dự phòng offline), và **KHÔNG bao giờ cache `/api/*`** (dữ liệu nhạy cảm + cần tươi). Đây là quyết định bảo mật quan trọng.

---

## 2.10. Property-based testing (hypothesis) — điểm nhấn test

- **Test thông thường (example-based)**: bạn tự nghĩ vài ví dụ cụ thể rồi assert.
- **Property-based testing**: bạn khai báo một **thuộc tính (property) luôn đúng**, rồi thư viện **hypothesis tự sinh hàng trăm input ngẫu nhiên** để cố phá vỡ nó.
- Dự án dùng hypothesis cho 3 thứ:
  1. `seconds_remaining` luôn nằm trong `[1, period]` với mọi thời điểm.
  2. **Round-trip** `parse_otpauth(build_otpauth(...))` bảo toàn mọi tham số (name/secret/digits/period/algorithm sinh ngẫu nhiên).
- Còn **test vector RFC 6238**: bảng cặp (thời điểm → mã) chính thức trong chuẩn, kiểm chứng implement đúng tuyệt đối với SHA1/256/512.

> 💡 Đây là điểm rất "kỹ sư": với code thuật toán, ngoài vài ví dụ còn có **chuẩn chính thức** + **fuzzing có cấu trúc**. Nêu được là một điểm cộng lớn.

---

## 2.11. Thuật ngữ khác

- **Vault**: "kho" — ở đây là file `vault.bin` chứa toàn bộ tài khoản đã mã hóa.
- **Dynamic truncation**: bước cắt digest HMAC thành mã ngắn (RFC 4226).
- **Master password (mật khẩu chủ)**: một mật khẩu duy nhất bảo vệ toàn bộ kho.
- **Round-trip**: làm A→B rồi B→A và kiểm tra ra lại đúng A (vd export→import, build→parse).
- **`os.chmod(0o600)`**: đặt quyền file chỉ chủ sở hữu đọc/ghi (bảo mật trên Unix).
- **CORS**: cơ chế cho phép trang web gọi API khác origin; dự án bật `allow_origins=["*"]` (an toàn vì chạy local).
