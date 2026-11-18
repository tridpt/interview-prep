# 3. System Architecture

## 3.1. High-level diagram

```
┌──────────────────────────── BROWSER (frontend) ─────────────────────────────┐
│  index.html  → lock screen, quick generate, list, backup, QR modal            │
│  app.js      → calls /api, renders list, countdown, QR scan (jsQR),           │
│                drag-to-reorder, light/dark theme                              │
│  sw.js       → PWA: network-first, NEVER caches /api                          │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                │ HTTP JSON (fetch) — 127.0.0.1 only
                                ▼
┌──────────────────────────── BACKEND — FastAPI (main.py) ────────────────────┐
│  /api/* endpoints  (status/setup/unlock/lock/generate/accounts/...)           │
│        │                         │                          │                 │
│        ▼                         ▼                          ▼                 │
│   totp.py                    store.py                    segno                │
│   (RFC 6238:                 (Session: vault encryption) (QR SVG)            │
│    HMAC, base32,             scrypt → Fernet                                   │
│    parse/build otpauth)      setup/unlock/lock/CRUD                            │
│        │                         │  reads config from config.py               │
└────────┼─────────────────────────┼────────────────────────────────────────────┘
         │                         │
   (no unlock needed)              ▼
         │                  storage/vault.bin  = { salt, data(ciphertext) }
         ▼                  Key (Fernet) is ONLY in RAM when unlocked
   Generate code from any secret
```

## 3.2. Layers & responsibilities

| Layer | File | Responsibility | Does NOT do |
|-------|------|----------------|-------------|
| Config | `config.py` | Vault path, scrypt params, auto-lock | No logic |
| TOTP algorithm | `totp.py` | Generate code (HMAC), normalize secret, parse/build otpauth | No vault/encryption |
| Encrypted vault | `store.py` | `Session`: setup/unlock/lock, CRUD, export/import, scrypt+Fernet encryption | No HTTP |
| API orchestration | `main.py` | REST endpoints, validation, calls totp/store, serves frontend | No direct encryption (via store) |
| UI | `frontend/*` | UI, API calls, render, QR scan | Doesn't hold secrets long-term |

**Key point for the interview:** **clean separation** — `totp.py` is pure algorithm (no state, no I/O), `store.py` is pure encryption/storage (no HTTP), `main.py` only orchestrates. So each part is **independently testable** (test_totp calls functions directly, test_store uses a temp vault, test_api uses TestClient).

## 3.3. Security model (the most important)

```
Master password ──scrypt(salt, N=32768,r=8,p=1)──▶ 32-byte key ──base64──▶ Fernet key
                                                                            │
        vault.bin = { salt, data }                                          │
                          │                                                 ▼
                     data = Fernet.encrypt( JSON list of accounts )
```

- **The key is NEVER written to disk** — it lives only in `Session.fernet` in RAM when unlocked.
- The vault file has only `salt` (can be public) + `data` (ciphertext). No password → can't derive the key → can't decrypt.
- **Unlock = authenticate**: `unlock()` tries `fernet.decrypt(data)`; `InvalidToken` → wrong password (`WrongPassword` → 401). There's no "password table" to compare against; successful decryption **is** the proof of the right password.
- **Encryption-at-rest**: the test `test_vault_encrypted_at_rest` checks the raw file **does not contain** plaintext secrets/names.

## 3.4. Data flow of main operations

**First time (setup):**
```
Frontend /status → initialized=false → show "Create master password"
→ POST /setup {password} → store.setup(): create salt, derive key, write empty vault → unlock
```

**Unlock & view codes:**
```
POST /unlock {password} → store.unlock(): derive key, try decrypt → on OK keep key in RAM
→ GET /accounts (every second) → load_accounts() decrypts → compute code + remaining per account
→ frontend renders the list + countdown rings
```

**Quick generate (NO unlock needed):**
```
POST /generate {secret} → totp.generate_code() → returns {code, remaining, period}
(doesn't touch the vault, doesn't save — to quickly test any secret)
```

**Add account:** `POST /accounts` → validate secret (`normalize_secret`) → `store.add_account()` → re-encrypt the whole vault → return the current code.

**Auto-lock:** each `GET /accounts` checks `is_unlocked()`; past 5 minutes of inactivity → `lock()` → returns 401 → frontend shows the lock screen again.

## 3.5. API endpoints (reference)

| Method | Endpoint | Unlock? | Function |
|--------|----------|:---:|----------|
| GET | `/api/status` | ✗ | State (setup / locked / idle remaining) |
| POST | `/api/setup` | ✗ | Create master password first time |
| POST | `/api/unlock` | ✗ | Unlock |
| POST | `/api/lock` | ✗ | Manual lock |
| POST | `/api/generate` | ✗ | Generate from a secret, no save |
| GET | `/api/accounts` | ✓ | List + current codes |
| POST | `/api/accounts` | ✓ | Add account |
| PATCH | `/api/accounts/{id}` | ✓ | Rename |
| PATCH | `/api/accounts/{id}/note` | ✓ | Set note |
| POST | `/api/accounts/reorder` | ✓ | Reorder |
| POST | `/api/accounts/import-uri` | ✓ | Add from `otpauth://` |
| GET | `/api/accounts/{id}/reveal` | ✓ | Get secret + otpauth + QR(SVG) |
| DELETE | `/api/accounts/{id}` | ✓ | Delete |
| POST | `/api/export` | ✓ | Export encrypted bundle |
| POST | `/api/import` | ✓ | Restore from bundle |
| GET | `/api/export-csv` | ✓ | Export CSV (plaintext secret) |
| POST | `/api/import-csv` | ✓ | Import from CSV |

## 3.6. Where does state live?

- **Sensitive state (decryption key)**: only in **server RAM** (`Session`), lost on server stop or auto-lock.
- **Persistent data**: the `vault.bin` file (encrypted).
- **Light client preference**: light/dark theme in `localStorage`. Accounts are NOT stored on the client.

## 3.7. Why is this architecture sound for a 2FA app?

- **Defense in depth**: encryption-at-rest + key-only-in-RAM + auto-lock + local-only.
- **Layered for testing**: algorithm/encryption/HTTP are independent → thoroughly tested per layer.
- **Reduced dependencies**: TOTP self-written from standard libraries (less attack surface); only use `cryptography` for the hard crypto.
- **Right level of simplicity**: single user, one global session — no complex login system needed because it's already local-only.
