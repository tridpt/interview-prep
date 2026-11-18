# 8. Cheat Sheet — Quick Review Before Go-Time

Skim for 5-10 minutes before the interview.

---

## 🎯 30-second pitch (memorize)
> OTP Vault: a locally-run **2FA (TOTP)** manager — paste a secret, get a 6-digit code changing every 30s per **RFC 6238**. Many accounts in an **encrypted vault** protected by a **master password**: key derived from the password via **scrypt**, encrypted with **Fernet (AES)**, **key only in RAM**, auto-locks after 5 minutes. **FastAPI** backend self-implements TOTP from standard libraries; **vanilla-JS (PWA)** frontend with QR scanning (jsQR). Tests include **RFC 6238 test vectors + property-based (hypothesis)**.

## 🧱 Tech stack
- **Backend**: FastAPI + uvicorn; `cryptography` (Scrypt + Fernet); `segno` (QR SVG); TOTP self-written with `hmac/hashlib/base64/struct`.
- **Frontend**: vanilla HTML/CSS/JS; jsQR (vendored); PWA (sw.js).
- **Tests**: pytest + hypothesis + httpx (TestClient).

## 🗂️ Files
| File | Role |
|---|---|
| config.py | vault path, scrypt params, auto-lock 300s |
| totp.py | TOTP (HMAC+truncation), normalize_secret, parse/build otpauth |
| store.py | Session: setup/unlock/lock, CRUD, export/import, scrypt+Fernet |
| main.py | FastAPI /api/*, serves frontend |
| frontend/app.js | API calls, render, countdown, QR scan, drag, theme |
| frontend/sw.js | PWA network-first, NEVER caches /api |

## 🔢 TOTP (know each step)
base32-decode the secret → counter = floor(time/30) → `struct.pack(">Q", counter)` → HMAC(key, counter, sha1) → **dynamic truncation** (offset = digest[-1]&0xF, read 4 bytes, `&0x7FFFFFFF`) → `% 10^digits` → zero-pad. SHA1/256/512. `seconds_remaining` ∈ [1, period].

## 🔐 Encryption (core model)
password → **scrypt(salt, N=32768)** → 32-byte key → **Fernet** (AES-128-CBC + HMAC-SHA256). `vault.bin = {salt, ciphertext}`. **Key ONLY in RAM** (Session.fernet). **unlock = try decrypt** (InvalidToken → wrong password). Forget password = lose data (no backdoor). File has no plaintext secret (tested).

## 🔁 Session & auto-lock
`is_unlocked()` = has fernet + not past 300s + file still exists. Operation → `touch()`. Expired / file deleted → `lock()` → API 401 → frontend shows lock screen.

## 🌐 API (unlock needed or not)
NOT needed: status, setup, unlock, lock, **generate** (code from a secret, no vault).
NEEDED: accounts (GET/POST/PATCH/DELETE), reorder, import-uri, reveal (QR+secret), export, import, export-csv, import-csv.
Status: 400 (bad request), 401 (locked/wrong password), 404 (not found).

## 🖼️ Frontend
`api()` wraps fetch (throws Error from detail). `setInterval(loadAccounts,1000)` refreshes codes + countdown. QR scanning via **jsQR on the client** (image never leaves the device, only otpauth sent). Drag → reorder. escapeHtml against XSS. SW **never caches /api**.

## 🧪 Tests (HIGHLIGHT)
- **RFC 6238 test vectors** (SHA1/256/512 × 4 moments) → exactly correct.
- **Property-based (hypothesis)**: seconds_remaining ∈ [1,period]; parse(build(...)) round-trip.
- **3 layers**: totp (algorithm) / store (vault) / api (TestClient).
- **conftest**: temp vault + reduced scrypt + reset session → never touches real data.
- **test_vault_encrypted_at_rest**: raw file doesn't leak the secret.
- Spec-driven: tests tagged "Req X.Y".

## 🛡️ Security (remember)
Key not on disk · local-only 127.0.0.1 · auto-lock · os.chmod(0o600) · CSV warns plaintext secret · SW no /api cache · escapeHtml.

## ⚖️ Trade-offs
No stored key → high security, forget password = lose data. Local-only → safe, no remote access. Self-written TOTP → fewer deps, must ensure correctness (offset by test vectors).

## 🚀 Future improvements
Rate-limit password attempts; raise scrypt N per hardware; encrypt per account; time-skew window on validation; multi-user if deployed for a team.

## 💬 Key terms
- **TOTP / HOTP / RFC 6238 / RFC 4226 / HMAC / base32 / dynamic truncation**
- **scrypt (KDF) / Fernet (AES) / salt / encryption-at-rest / zero-knowledge**
- **Session / auto-lock / master password**
- **FastAPI / Pydantic / StaticFiles / 401**
- **jsQR / getUserMedia / Canvas / PWA / network-first**
- **pytest / hypothesis / property-based / RFC test vectors / spec-driven**

---

### ✅ Pre-interview checklist
- [ ] 30-second pitch.
- [ ] Explain TOTP step by step + why self-written.
- [ ] The "no stored key" model + unlock authentication.
- [ ] Auto-lock & edge-case handling.
- [ ] RFC test vectors + property-based.
- [ ] Client QR scanning & SW not caching /api.
- [ ] 2 limitations + fixes.
- [ ] 2 questions for the interviewer.

**Breathe. You understand this project. Tell your story with confidence. 🍀**
