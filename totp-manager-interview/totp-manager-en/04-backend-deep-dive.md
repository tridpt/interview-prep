# 4. Backend Deep Dive (file by file, function by function)

> Written after directly reading 100% of the backend. Read alongside the real code in `backend/`.

---

## 4.1. `config.py` — Configuration

Very short, just constants:
```python
BASE_DIR = Path(__file__).resolve().parent.parent
STORAGE_DIR = BASE_DIR / "storage"      # auto-mkdir
VAULT_PATH = STORAGE_DIR / "vault.bin"  # encrypted vault file
SCRYPT_N = 2**15  # 32768   ── scrypt param (brute-force resistance)
SCRYPT_R = 8
SCRYPT_P = 1
SCRYPT_DKLEN = 32           # 32 bytes → Fernet key
AUTO_LOCK_SECONDS = 300     # auto-lock after 5 minutes
```
- High `SCRYPT_N` → key derivation is **intentionally slow** (anti password-guessing). `DKLEN=32` because Fernet needs a 32-byte key.

---

## 4.2. `totp.py` — TOTP algorithm (RFC 6238)

A **pure-algorithm** module, no state, no I/O. Standard library only.

**`TOTPError(ValueError)`**: error for bad secret/params.

**`normalize_secret(secret)`**:
```python
cleaned = secret.strip().replace(" ", "").replace("-", "").upper()
pad = (-len(cleaned)) % 8           # base32 needs a multiple of 8
cleaned += "=" * pad
base64.b32decode(cleaned, casefold=True)   # validate, invalid → TOTPError
return cleaned
```
Handles messy user-copied secrets (spaces/dashes), adds `=` padding, validates base32. **Idempotent**.

**`generate_code(secret, *, digits=6, period=30, algorithm="SHA1", at=None)`** — the heart:
```python
key = base64.b32decode(normalize_secret(secret), casefold=True)
algo = _ALGOS[algorithm.upper()]          # SHA1/256/512, invalid → TOTPError
counter = int((at or time.time()) // period)
msg = struct.pack(">Q", counter)          # 8 bytes big-endian
digest = hmac.new(key, msg, algo).digest()
offset = digest[-1] & 0x0F                 # dynamic truncation (RFC 4226)
code_int = struct.unpack(">I", digest[offset:offset+4])[0] & 0x7FFFFFFF
return str(code_int % 10**digits).zfill(digits)
```
- The `at` parameter allows **testing with a fixed time** (e.g. 59, 1111111109) → matches the RFC vectors.

**`seconds_remaining(period=30, at=None)`**: `period - int(now % period)` → always within **[1, period]** (never 0; property-based tested).

**`parse_otpauth(uri)`**: splits `otpauth://totp/...?secret=...`; only accepts `totp` (rejects `hotp`); missing secret → error; builds the name from `issuer`+label.

**`build_otpauth(name, secret, ...)`**: builds the reverse URI `otpauth://totp/{label}?secret=...&digits=...` to make a QR. `parse(build(...))` round-trips losslessly (hypothesis test).

---

## 4.3. `store.py` — Encrypted vault (`Session`)

**Exceptions**: `VaultLocked` (vault locked), `WrongPassword` (wrong password).

**`_derive_fernet(password, salt)`** — the password → key bridge:
```python
kdf = Scrypt(salt=salt, length=32, n=SCRYPT_N, r=SCRYPT_R, p=SCRYPT_P)
raw = kdf.derive(password.encode())
return Fernet(base64.urlsafe_b64encode(raw))
```

**`is_initialized()`**: whether the vault exists (`VAULT_PATH.exists()`).

**`_read_vault_file()` / `_write_vault_file(salt, fernet, accounts)`**: read/write the `{salt(b64), data(token)}` file. Write: `json → fernet.encrypt → b64`. After writing, `os.chmod(0o600)` (owner-only read/write).

**`class Session`** (holds the key in RAM):
- Fields: `fernet`, `salt`, `last_activity`.
- **`is_unlocked()`**: True only when there's a `fernet` **and** not past `AUTO_LOCK_SECONDS` **and** the vault file still exists (if the file is deleted → auto `lock()`, avoiding a 500). A notable **hardening**.
- `touch()` updates `last_activity`; `lock()` clears `fernet`/`salt`; `remaining_idle()` is the seconds left before auto-lock.
- **`setup(password)`**: create a random 16-byte salt, derive the key, write an empty vault `[]`, unlock.
- **`unlock(password)`**: read the salt from the file, derive the key, **try `fernet.decrypt(data)`** → `InvalidToken` raises `WrongPassword`. (Successful decryption = correct password.)
- `_require()`: the gatekeeper — if not unlocked raise `VaultLocked`, else `touch()` then return `fernet`.
- **`load_accounts()`**: `_require()` → decrypt `data` → `json.loads` → list.
- `get_account(id)`, `_save_accounts(accounts)` (re-encrypt & rewrite).
- **`add_account(name, secret, ...)`**: create `id = uuid4().hex[:12]`, append, save. **`delete_account`**, **`rename_account`** (empty → "Tài khoản"), **`set_note`**, **`reorder_accounts(ordered_ids)`** (order by given ids, junk ids skipped, unmentioned items kept at the end — **no data loss**).
- **`export_backup(password)`**: create a **separate salt** + key from a **separate password** → encrypt the list → bundle `{type:"otp-vault-backup", version:1, salt, data}`. Self-contained, restorable on another machine.
- **`import_backup(password, backup)`**: check `type`, derive key from the bundle's salt, decrypt (wrong → `WrongPassword`); **merge** into the vault, **skip duplicates** (same name+secret), **assign new ids**. Returns the added count.
- **`import_entries(entries)`** (for CSV): each entry needs a `secret`; **validate via `normalize_secret`** (invalid → skip), skip duplicates, assign new ids.

`session = Session()` at the bottom — a **module-global** shared instance (local app, single user).

---

## 4.4. `main.py` — FastAPI

**Setup**: `app = FastAPI(...)`, adds CORS `allow_origins=["*"]` (safe because local). Imports `segno` in try/except (if missing, only QR is disabled).

**Pydantic schemas**: `GenerateRequest`, `PasswordRequest`, `AddAccountRequest`, `ImportUriRequest`, `ImportBackupRequest`, `ImportCsvRequest`, `RenameRequest`, `NoteRequest`, `ReorderRequest` — FastAPI auto-validates JSON against these.

**Helpers:**
- `_ensure_unlocked()`: if `not session.is_unlocked()` → raise `HTTPException(401)`. The gatekeeper for every endpoint needing unlock.
- `_code_payload(account)`: computes the current code + `remaining` for one account; **catches `TOTPError`** → returns `{code:None, error:...}` instead of crashing (one bad-secret account doesn't break the whole list).

**Endpoints (1-1 with store):**
- `GET /api/status` → `{initialized, locked, auto_lock_seconds, idle_remaining}`.
- `POST /api/setup` → blocks if already set up (400), password <6 (400), then `session.setup`.
- `POST /api/unlock` → `WrongPassword` → 401.
- `POST /api/lock`.
- `POST /api/generate` → `totp.generate_code`, bad secret → 400. **No unlock needed.**
- `GET /api/accounts` → `_ensure_unlocked` → list of `_code_payload`.
- `POST /api/accounts` → validate `normalize_secret` (400 if invalid) → `add_account`.
- `POST /api/accounts/import-uri` → `parse_otpauth` → `add_account`.
- `DELETE /api/accounts/{id}` → 404 if missing.
- `PATCH /api/accounts/{id}` (rename), `PATCH .../note`, `POST .../reorder`.
- `GET /api/accounts/{id}/reveal` → `build_otpauth` + `segno.make(uri).svg_inline(...)` (SVG scales thanks to `omitsize=True`) + secret.
- `POST /api/export` (backup password ≥6) → `export_backup`.
- `POST /api/import` → `import_backup` (401/400 per error).
- `GET /api/export-csv` → build CSV with `csv.writer`, prepend a **BOM `\ufeff`** so Excel opens Vietnamese correctly; warns the secret is plaintext.
- `POST /api/import-csv` → `csv.DictReader`, **requires a `secret` column** (400 if missing), normalize column names to lowercase, call `import_entries`.

**End of file**: `app.mount("/", StaticFiles(directory=frontend_dir, html=True))` — serves the static frontend (placed **last** so it doesn't shadow `/api`).

---

## 4.5. The test suite (tests/) — the highlight

**`conftest.py`**: the `isolated_vault` fixture (`autouse=True`) → **every test** uses a vault in a **temp** directory (`monkeypatch VAULT_PATH`), **reduces `SCRYPT_N` to 2^10** for fast tests, and resets `session.lock()` before/after each test → **never touches real data**.

**`test_totp.py`** (Req 2-5):
- `test_rfc6238_vectors`: the **official RFC 6238 test vectors** (SHA1/256/512 × 4 timestamps) → verifies the implementation is exactly correct.
- Code length per `digits`, same period same code, bad algorithm → error.
- `normalize_secret`: strip/uppercase/padding, empty → error, bad base32 → error, idempotent.
- `parse/build otpauth`: valid, non-otpauth → error, missing secret → error, hotp → error.
- **`test_build_parse_roundtrip`** (hypothesis): random name/seed/digits/period/algorithm → `parse(build(...))` preserves them.
- **`test_seconds_remaining_bounds`** (hypothesis, 200 examples): always within [1, period].

**`test_store.py`** (Req 6-9): setup/unlock/lock, wrong password stays locked, **auto-lock** (simulating an expired `last_activity`), activity pushes back the deadline, **vault file deleted → treated as locked** (no 500), CRUD, **`test_vault_encrypted_at_rest`** (raw file has no plaintext secret/name), export/import round-trip + wrong password + skip duplicates + new ids, rename/reorder/note, import_entries (CSV) skips bad/duplicate.

**`test_api.py`** (Req 10-11): via `TestClient` — uninitialized status, `/accounts` while locked → 401, a **parametrized table where all unlock-needing endpoints return 401**, short password → 400, wrong unlock password → 401, setup→add→list flow, lock→unlock, generate without unlock, reveal returns QR+secret, export/import CSV round-trip keeping notes.

> 💡 This suite covers the **algorithm (RFC standard + property-based)**, **security (encryption-at-rest, auto-lock, the 401 rule)**, and **end-to-end API** — well worth showing off in an interview.

---

## 4.6. Summary of backend technical decisions

| Decision | Reason |
|----------|--------|
| TOTP self-written from `hmac`/`hashlib` | Fewer dependencies, full understanding, verified by RFC vectors |
| scrypt + Fernet, key only in RAM | Encryption-at-rest, "no stored key", forget password = lose data |
| `unlock` = attempt to decrypt | No need to store a password hash; successful decryption = correct password |
| Global `Session` + auto-lock | Local single user; reduces risk of leaving it unlocked |
| Separate `totp`/`store`/`main` | Independent per-layer testing |
| `_code_payload` catches TOTPError | One bad account doesn't crash the list |
| `generate` needs no unlock | Convenient quick test, doesn't touch the vault |
| Isolate a temp vault in tests | Never touches real data |
