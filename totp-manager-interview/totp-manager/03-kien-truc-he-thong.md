# 3. Kiến trúc hệ thống

## 3.1. Sơ đồ tổng thể

```
┌──────────────────────────── TRÌNH DUYỆT (frontend) ─────────────────────────┐
│  index.html  → màn khóa, sinh mã nhanh, danh sách, sao lưu, modal QR          │
│  app.js      → gọi /api, render danh sách, đếm ngược, quét QR (jsQR),         │
│                kéo-thả sắp xếp, theme sáng/tối                                 │
│  sw.js       → PWA: network-first, KHÔNG cache /api                           │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                │ HTTP JSON (fetch) — chỉ 127.0.0.1
                                ▼
┌──────────────────────────── BACKEND — FastAPI (main.py) ────────────────────┐
│  Các endpoint /api/*  (status/setup/unlock/lock/generate/accounts/...)        │
│        │                         │                          │                 │
│        ▼                         ▼                          ▼                 │
│   totp.py                    store.py                    segno                │
│   (RFC 6238:                 (Session: mã hóa kho)       (QR SVG)             │
│    HMAC, base32,             scrypt → Fernet                                   │
│    parse/build otpauth)      setup/unlock/lock/CRUD                            │
│        │                         │  đọc cấu hình từ config.py                  │
└────────┼─────────────────────────┼────────────────────────────────────────────┘
         │                         │
   (không cần mở khóa)             ▼
         │                  storage/vault.bin  = { salt, data(ciphertext) }
         ▼                  Khóa (Fernet) CHỈ ở RAM khi đã mở khóa
   Sinh mã từ secret bất kỳ
```

## 3.2. Phân lớp & trách nhiệm

| Lớp | File | Trách nhiệm | Không làm gì |
|-----|------|-------------|--------------|
| Cấu hình | `config.py` | Đường dẫn vault, tham số scrypt, auto-lock | Không logic |
| Thuật toán TOTP | `totp.py` | Sinh mã (HMAC), chuẩn hóa secret, parse/build otpauth | Không đụng kho/mã hóa |
| Kho mã hóa | `store.py` | `Session`: setup/unlock/lock, CRUD, export/import, mã hóa scrypt+Fernet | Không biết HTTP |
| Điều phối API | `main.py` | Endpoint REST, validate, gọi totp/store, phục vụ frontend | Không tự mã hóa (qua store) |
| Giao diện | `frontend/*` | UI, gọi API, render, quét QR | Không giữ secret lâu dài |

**Điểm cốt lõi để nói khi phỏng vấn:** **tách bạch sạch sẽ** — `totp.py` thuần thuật toán (không state, không I/O), `store.py` thuần mã hóa/lưu trữ (không HTTP), `main.py` chỉ điều phối. Nhờ vậy mỗi phần **test độc lập được** (test_totp gọi thẳng hàm, test_store dùng vault tạm, test_api dùng TestClient).

## 3.3. Mô hình bảo mật (quan trọng nhất)

```
Mật khẩu chủ ──scrypt(salt, N=32768,r=8,p=1)──▶ khóa 32 byte ──base64──▶ khóa Fernet
                                                                          │
        vault.bin = { salt, data }                                        │
                          │                                               ▼
                     data = Fernet.encrypt( JSON danh sách tài khoản )
```

- **Khóa KHÔNG bao giờ ghi ra đĩa** — chỉ nằm trong `Session.fernet` ở RAM khi mở khóa.
- File vault chỉ có `salt` (công khai được) + `data` (ciphertext). Không mật khẩu → không dẫn được khóa → không giải mã.
- **Mở khóa = xác thực**: `unlock()` thử `fernet.decrypt(data)`; nếu `InvalidToken` → sai mật khẩu (`WrongPassword` → 401). Không có "bảng mật khẩu" để so sánh; chính việc giải mã thành công **là** bằng chứng đúng mật khẩu.
- **Mã hóa-tại-chỗ (at rest)**: test `test_vault_encrypted_at_rest` kiểm tra file thô **không chứa** secret/tên dạng văn bản rõ.

## 3.4. Luồng dữ liệu các thao tác chính

**Lần đầu (setup):**
```
Frontend /status → initialized=false → hiện màn "Tạo mật khẩu chủ"
→ POST /setup {password} → store.setup(): tạo salt, dẫn khóa, ghi vault rỗng → mở khóa
```

**Mở khóa & xem mã:**
```
POST /unlock {password} → store.unlock(): dẫn khóa, thử decrypt → OK thì giữ khóa ở RAM
→ GET /accounts (mỗi giây) → load_accounts() giải mã → mỗi tài khoản tính code + remaining
→ frontend render danh sách + vòng đếm ngược
```

**Sinh mã nhanh (KHÔNG cần mở khóa):**
```
POST /generate {secret} → totp.generate_code() → trả {code, remaining, period}
(không đụng kho, không lưu — dùng để thử nhanh một secret bất kỳ)
```

**Thêm tài khoản:** `POST /accounts` → validate secret (`normalize_secret`) → `store.add_account()` → mã hóa lại cả kho → trả mã hiện tại.

**Auto-lock:** mỗi `GET /accounts` kiểm `is_unlocked()`; quá 5 phút không thao tác → `lock()` → trả 401 → frontend hiện lại màn khóa.

## 3.5. Các endpoint API (bảng tra cứu)

| Method | Endpoint | Cần mở khóa? | Chức năng |
|--------|----------|:---:|-----------|
| GET | `/api/status` | ✗ | Trạng thái (đã setup / đang khóa / idle còn lại) |
| POST | `/api/setup` | ✗ | Tạo mật khẩu chủ lần đầu |
| POST | `/api/unlock` | ✗ | Mở khóa |
| POST | `/api/lock` | ✗ | Khóa thủ công |
| POST | `/api/generate` | ✗ | Sinh mã từ secret, không lưu |
| GET | `/api/accounts` | ✓ | Danh sách + mã hiện tại |
| POST | `/api/accounts` | ✓ | Thêm tài khoản |
| PATCH | `/api/accounts/{id}` | ✓ | Đổi tên |
| PATCH | `/api/accounts/{id}/note` | ✓ | Đặt ghi chú |
| POST | `/api/accounts/reorder` | ✓ | Sắp xếp lại |
| POST | `/api/accounts/import-uri` | ✓ | Thêm từ `otpauth://` |
| GET | `/api/accounts/{id}/reveal` | ✓ | Lấy secret + otpauth + QR(SVG) |
| DELETE | `/api/accounts/{id}` | ✓ | Xóa |
| POST | `/api/export` | ✓ | Xuất gói mã hóa |
| POST | `/api/import` | ✓ | Khôi phục từ gói |
| GET | `/api/export-csv` | ✓ | Xuất CSV (secret dạng rõ) |
| POST | `/api/import-csv` | ✓ | Nhập từ CSV |

## 3.6. Trạng thái nằm ở đâu?

- **Trạng thái nhạy cảm (khóa giải mã)**: chỉ ở **RAM server** (`Session`), mất khi tắt server hoặc auto-lock.
- **Dữ liệu lâu dài**: file `vault.bin` (mã hóa).
- **Tùy chọn nhẹ phía client**: theme sáng/tối ở `localStorage`. Tài khoản KHÔNG lưu ở client.

## 3.7. Vì sao kiến trúc này hợp lý cho một app 2FA?

- **Bảo mật theo chiều sâu**: mã hóa-tại-chỗ + khóa chỉ ở RAM + auto-lock + local-only.
- **Tách lớp để test**: thuật toán/mã hóa/HTTP độc lập → test kỹ từng phần.
- **Giảm phụ thuộc**: TOTP tự viết từ thư viện chuẩn (ít bề mặt tấn công); chỉ dùng `cryptography` cho phần mật mã khó.
- **Đơn giản đúng mức**: một người dùng, một phiên toàn cục — không cần hệ thống đăng nhập phức tạp vì đã local-only.
