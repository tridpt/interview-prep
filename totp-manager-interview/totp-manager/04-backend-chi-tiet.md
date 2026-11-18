# 4. Backend chi tiết (từng file, từng hàm)

> Viết sau khi đọc trực tiếp 100% backend. Đọc kèm code thật trong `backend/`.

---

## 4.1. `config.py` — Cấu hình

Rất ngắn, chỉ định nghĩa hằng số:
```python
BASE_DIR = Path(__file__).resolve().parent.parent
STORAGE_DIR = BASE_DIR / "storage"      # tự mkdir
VAULT_PATH = STORAGE_DIR / "vault.bin"  # file kho mã hóa
SCRYPT_N = 2**15  # 32768   ── tham số scrypt (chống brute-force)
SCRYPT_R = 8
SCRYPT_P = 1
SCRYPT_DKLEN = 32           # 32 byte → khóa Fernet
AUTO_LOCK_SECONDS = 300     # tự khóa sau 5 phút
```
- `SCRYPT_N` cao → dẫn khóa **chậm có chủ đích** (chống dò mật khẩu). `DKLEN=32` vì Fernet cần khóa 32 byte.

---

## 4.2. `totp.py` — Thuật toán TOTP (RFC 6238)

Module **thuần thuật toán**, không state, không I/O. Chỉ dùng thư viện chuẩn.

**`TOTPError(ValueError)`**: lỗi khi secret/tham số sai.

**`normalize_secret(secret)`**:
```python
cleaned = secret.strip().replace(" ", "").replace("-", "").upper()
pad = (-len(cleaned)) % 8           # base32 cần độ dài bội số 8
cleaned += "=" * pad
base64.b32decode(cleaned, casefold=True)   # validate, sai → TOTPError
return cleaned
```
Xử lý secret người dùng copy lộn xộn (dấu cách/gạch), thêm padding `=`, xác thực base32. **Idempotent**.

**`generate_code(secret, *, digits=6, period=30, algorithm="SHA1", at=None)`** — trái tim:
```python
key = base64.b32decode(normalize_secret(secret), casefold=True)
algo = _ALGOS[algorithm.upper()]          # SHA1/256/512, sai → TOTPError
counter = int((at or time.time()) // period)
msg = struct.pack(">Q", counter)          # 8 byte big-endian
digest = hmac.new(key, msg, algo).digest()
offset = digest[-1] & 0x0F                 # dynamic truncation (RFC 4226)
code_int = struct.unpack(">I", digest[offset:offset+4])[0] & 0x7FFFFFFF
return str(code_int % 10**digits).zfill(digits)
```
- Tham số `at` cho phép **test với thời điểm cố định** (vd 59, 1111111109) → khớp test vector RFC.

**`seconds_remaining(period=30, at=None)`**: `period - int(now % period)` → luôn trong **[1, period]** (không bao giờ 0; có test property-based).

**`parse_otpauth(uri)`**: tách `otpauth://totp/...?secret=...`; chỉ chấp nhận `totp` (từ chối `hotp`); thiếu secret → lỗi; ghép tên từ `issuer`+label.

**`build_otpauth(name, secret, ...)`**: dựng lại URI `otpauth://totp/{label}?secret=...&digits=...` để tạo QR. `parse(build(...))` round-trip bảo toàn tham số (test hypothesis).

---

## 4.3. `store.py` — Kho mã hóa (`Session`)

**Exceptions**: `VaultLocked` (kho đang khóa), `WrongPassword` (sai mật khẩu).

**`_derive_fernet(password, salt)`** — cầu nối mật khẩu → khóa:
```python
kdf = Scrypt(salt=salt, length=32, n=SCRYPT_N, r=SCRYPT_R, p=SCRYPT_P)
raw = kdf.derive(password.encode())
return Fernet(base64.urlsafe_b64encode(raw))
```

**`is_initialized()`**: vault tồn tại chưa (`VAULT_PATH.exists()`).

**`_read_vault_file()` / `_write_vault_file(salt, fernet, accounts)`**: đọc/ghi file `{salt(b64), data(token)}`. Ghi: `json → fernet.encrypt → b64`. Sau ghi gọi `os.chmod(0o600)` (chỉ chủ đọc/ghi).

**`class Session`** (giữ khóa ở RAM):
- Trường: `fernet`, `salt`, `last_activity`.
- **`is_unlocked()`**: True chỉ khi có `fernet` **và** chưa quá `AUTO_LOCK_SECONDS` **và** file vault còn tồn tại (nếu file bị xóa → tự `lock()`, tránh lỗi 500). Đây là **hardening** đáng nói.
- `touch()` cập nhật `last_activity`; `lock()` xóa `fernet`/`salt`; `remaining_idle()` số giây còn lại trước auto-lock.
- **`setup(password)`**: tạo salt 16 byte ngẫu nhiên, dẫn khóa, ghi vault rỗng `[]`, mở khóa.
- **`unlock(password)`**: đọc salt từ file, dẫn khóa, **thử `fernet.decrypt(data)`** → `InvalidToken` thì raise `WrongPassword`. (Giải mã thành công = mật khẩu đúng.)
- `_require()`: gác cổng — nếu chưa mở khóa raise `VaultLocked`, ngược lại `touch()` rồi trả `fernet`.
- **`load_accounts()`**: `_require()` → giải mã `data` → `json.loads` → list.
- `get_account(id)`, `_save_accounts(accounts)` (mã hóa & ghi lại).
- **`add_account(name, secret, ...)`**: tạo `id = uuid4().hex[:12]`, append, lưu. **`delete_account`**, **`rename_account`** (rỗng → "Tài khoản"), **`set_note`**, **`reorder_accounts(ordered_ids)`** (sắp theo id cho trước, id rác bị bỏ qua, mục không nhắc tới giữ ở cuối — **không mất dữ liệu**).
- **`export_backup(password)`**: tạo **salt riêng** + khóa từ mật khẩu **riêng** → mã hóa danh sách → gói `{type:"otp-vault-backup", version:1, salt, data}`. Tự chứa salt nên khôi phục được trên máy khác.
- **`import_backup(password, backup)`**: kiểm `type`, dẫn khóa từ salt trong gói, giải mã (sai → `WrongPassword`); **gộp** vào kho, **bỏ qua trùng** (cùng name+secret), **cấp id mới** cho mục thêm. Trả số mục thêm.
- **`import_entries(entries)`** (cho CSV): mỗi entry cần `secret`; **validate bằng `normalize_secret`** (sai → bỏ qua), bỏ qua trùng, cấp id mới.

`session = Session()` ở cuối — **biến module toàn cục** dùng chung (app local, một người dùng).

---

## 4.4. `main.py` — FastAPI

**Khởi tạo**: `app = FastAPI(...)`, thêm CORS `allow_origins=["*"]` (an toàn vì local). Import `segno` trong try/except (thiếu thì chỉ tắt tính năng QR).

**Pydantic schemas**: `GenerateRequest`, `PasswordRequest`, `AddAccountRequest`, `ImportUriRequest`, `ImportBackupRequest`, `ImportCsvRequest`, `RenameRequest`, `NoteRequest`, `ReorderRequest` — FastAPI tự validate JSON theo các class này.

**Helpers:**
- `_ensure_unlocked()`: nếu `not session.is_unlocked()` → raise `HTTPException(401)`. Gác cổng mọi endpoint cần mở khóa.
- `_code_payload(account)`: tính mã hiện tại + `remaining` cho một tài khoản; **bắt `TOTPError`** → trả `{code:None, error:...}` thay vì sập (một tài khoản secret hỏng không làm hỏng cả danh sách).

**Endpoints (ánh xạ 1-1 với store):**
- `GET /api/status` → `{initialized, locked, auto_lock_seconds, idle_remaining}`.
- `POST /api/setup` → chặn nếu đã setup (400), mật khẩu <6 ký tự (400), rồi `session.setup`.
- `POST /api/unlock` → `WrongPassword` → 401.
- `POST /api/lock`.
- `POST /api/generate` → `totp.generate_code`, secret sai → 400. **Không cần mở khóa.**
- `GET /api/accounts` → `_ensure_unlocked` → danh sách `_code_payload`.
- `POST /api/accounts` → validate `normalize_secret` (400 nếu sai) → `add_account`.
- `POST /api/accounts/import-uri` → `parse_otpauth` → `add_account`.
- `DELETE /api/accounts/{id}` → 404 nếu không có.
- `PATCH /api/accounts/{id}` (đổi tên), `PATCH .../note`, `POST .../reorder`.
- `GET /api/accounts/{id}/reveal` → `build_otpauth` + `segno.make(uri).svg_inline(...)` (SVG co giãn nhờ `omitsize=True`) + secret.
- `POST /api/export` (mật khẩu sao lưu ≥6) → `export_backup`.
- `POST /api/import` → `import_backup` (401/400 theo lỗi).
- `GET /api/export-csv` → dựng CSV bằng `csv.writer`, thêm **BOM `\ufeff`** để Excel mở đúng tiếng Việt; cảnh báo secret dạng rõ.
- `POST /api/import-csv` → `csv.DictReader`, **bắt buộc có cột `secret`** (400 nếu thiếu), chuẩn hóa tên cột về chữ thường, gọi `import_entries`.

**Cuối file**: `app.mount("/", StaticFiles(directory=frontend_dir, html=True))` — phục vụ frontend tĩnh (đặt **cuối** để không che `/api`).

---

## 4.5. Bộ test (tests/) — điểm nhấn

**`conftest.py`**: fixture `isolated_vault` (`autouse=True`) → **mọi test** dùng vault ở thư mục **tạm** (`monkeypatch VAULT_PATH`), **giảm `SCRYPT_N` xuống 2^10** cho test nhanh, và reset `session.lock()` trước/sau mỗi test → **không bao giờ đụng dữ liệu thật**.

**`test_totp.py`** (Req 2-5):
- `test_rfc6238_vectors`: bảng **test vector RFC 6238 chính thức** (SHA1/256/512 × 4 mốc thời gian) → kiểm chứng implement đúng tuyệt đối.
- Độ dài mã theo `digits`, cùng chu kỳ cùng mã, thuật toán sai → lỗi.
- `normalize_secret`: strip/uppercase/padding, rỗng → lỗi, base32 sai → lỗi, idempotent.
- `parse/build otpauth`: hợp lệ, không-otpauth → lỗi, thiếu secret → lỗi, hotp → lỗi.
- **`test_build_parse_roundtrip`** (hypothesis): sinh ngẫu nhiên name/seed/digits/period/algorithm → `parse(build(...))` bảo toàn.
- **`test_seconds_remaining_bounds`** (hypothesis, 200 ví dụ): luôn trong [1, period].

**`test_store.py`** (Req 6-9): setup/unlock/lock, sai mật khẩu giữ khóa, **auto-lock** (giả lập `last_activity` quá hạn), thao tác dời mốc auto-lock, **file vault bị xóa → coi như khóa** (không 500), CRUD, **`test_vault_encrypted_at_rest`** (file thô không chứa secret/tên rõ), export/import round-trip + sai mật khẩu + bỏ trùng + cấp id mới, rename/reorder/note, import_entries (CSV) bỏ mục sai/trùng.

**`test_api.py`** (Req 10-11): qua `TestClient` — status chưa init, `/accounts` khi khóa → 401, **bảng parametrize các endpoint cần mở khóa đều 401**, mật khẩu ngắn → 400, sai mật khẩu unlock → 401, luồng setup→add→list, lock→unlock, generate không cần mở khóa, reveal trả QR+secret, export/import CSV round-trip giữ cả note.

> 💡 Bộ test này phủ cả **thuật toán (chuẩn RFC + property-based)**, **bảo mật (mã hóa-tại-chỗ, auto-lock, quy tắc 401)**, và **API end-to-end** — rất đáng để khoe khi phỏng vấn.

---

## 4.6. Tổng kết quyết định kỹ thuật backend

| Quyết định | Lý do |
|-----------|-------|
| TOTP tự viết từ `hmac`/`hashlib` | Giảm phụ thuộc, hiểu rõ thuật toán, kiểm chứng bằng RFC vectors |
| scrypt + Fernet, khóa chỉ ở RAM | Mã hóa-tại-chỗ, "không lưu khóa", quên mật khẩu = mất dữ liệu |
| `unlock` = thử giải mã | Không cần lưu hash mật khẩu riêng; giải mã OK = đúng mật khẩu |
| `Session` toàn cục + auto-lock | Local một người dùng; giảm rủi ro để mở khóa quên |
| Tách `totp`/`store`/`main` | Test độc lập từng lớp |
| `_code_payload` bắt TOTPError | Một tài khoản hỏng không làm sập cả danh sách |
| `generate` không cần mở khóa | Tiện thử nhanh secret, không đụng kho |
| Cô lập vault tạm khi test | Không bao giờ đụng dữ liệu thật |
