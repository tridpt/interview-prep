# 2. Foundational Concepts (theory)

This file explains **every term** in the project. It's a **security/crypto** project, so concepts matter a lot — mastering them scores big.

---

## 2.1. 2FA, OTP, TOTP, HOTP

- **2FA (Two-Factor Authentication)**: besides a password, login needs a "second factor" (a constantly-changing code).
- **OTP (One-Time Password)**: a single-use password.
- **HOTP (HMAC-based OTP, RFC 4226)**: an OTP based on an incrementing **counter**.
- **TOTP (Time-based OTP, RFC 6238)**: a variant of HOTP where the **counter = time**. Specifically `counter = floor(current_time / period)`, with `period` usually 30 seconds. Since both server and app know the time, they compute the **same code** without communicating.

➡️ This project generates **TOTP** — exactly the Google Authenticator/Authy standard.

---

## 2.2. How TOTP works (the core of the project)

This is the part you **must know cold**. The steps in `totp.py` → `generate_code`:

1. **Decode the secret**: the secret is a **base32** string (A-Z, 2-7). Decode to bytes → this is the **HMAC key**.
2. **Compute the counter**: `counter = int(time // period)` (number of 30-second periods since 1970).
3. **Pack the counter**: `struct.pack(">Q", counter)` → 8 bytes big-endian.
4. **HMAC**: `hmac.new(key, counter_bytes, sha1).digest()` → a 20-byte digest (SHA1).
5. **Dynamic truncation (RFC 4226)**: take the last 4 bits of the digest as `offset`, read 4 bytes from `offset`, drop the sign bit (`& 0x7FFFFFFF`) → a 31-bit number.
6. **Get the code**: `number % 10^digits` then zero-pad to `digits` (default 6) → "492039".

**Why the truncation step?** To turn a long digest into a short, easy-to-type number that's still well-distributed and hard to guess.

> 💡 If asked "how is TOTP different from a normal password", say: it's **never stored/transmitted**, both sides compute it from secret + time; the code lives only ~30s so a leak expires quickly.

---

## 2.3. HMAC & hash functions (SHA1/256/512)

- **Hash function**: turns any data into a fixed-size digest, one-way. SHA1 → 20 bytes, SHA256 → 32, SHA512 → 64.
- **HMAC (Hash-based Message Authentication Code)**: using a hash **with a secret key** to make an authentication code. TOTP uses HMAC(secret, counter).
- The project supports 3 algorithms via the `_ALGOS = {SHA1, SHA256, SHA512}` dict; default SHA1 (the most common authenticator standard).

---

## 2.4. Base32 & secret normalization

- **Base32**: encoding binary with 32 characters (A-Z, 2-7). 2FA uses base32 because it's easy to type (no 0/O, 1/l confusion).
- Users often copy secrets with **spaces/dashes** (e.g. `JBSW Y3DP EHPK 3PXP`). `normalize_secret` handles this: strip whitespace/dashes, **uppercase**, **add `=` padding** to a multiple of 8 (base32's requirement), then try `b32decode` to validate; invalid → `TOTPError`.
- **Idempotent**: normalizing twice gives the same result (tested) — important so re-normalizing doesn't corrupt a clean secret.

---

## 2.5. otpauth:// URI & QR code

- When enabling 2FA, services usually give a **QR code** containing `otpauth://totp/Issuer:account?secret=ABC&issuer=...&digits=6&period=30&algorithm=SHA1`.
- `parse_otpauth` splits this into parameters (name/secret/digits/period/algorithm); only accepts type `totp` (rejects `hotp`).
- `build_otpauth` does the reverse — builds a URI to **recreate the QR** (to add to another app). `segno` turns the URI into a QR SVG.
- The frontend uses **jsQR** to **read** QR from camera/image in the browser (no image is sent to the server).

---

## 2.6. Vault encryption: scrypt (KDF) + Fernet (AES)

This is the **core security model** — know it well.

- **KDF (Key Derivation Function)**: a function that turns a **password** (human-memorable) into an **encryption key** (strong random bytes). The project uses **scrypt** — a KDF that's **brute-force-resistant** because it's intentionally **memory + CPU heavy** (params `N=32768, r=8, p=1`, outputs a 32-byte key).
- **Salt**: 16 random bytes stored with the vault; same password but different salt → different key → defeats lookup tables (rainbow tables).
- **Fernet**: a safe symmetric-encryption "recipe" from the `cryptography` library — internally **AES-128-CBC + HMAC-SHA256** (encrypts **and authenticates integrity**). The Fernet key = `base64(scrypt_output_32bytes)`.
- **The "no stored key" model**: the Fernet key exists only in **server RAM** when unlocked; `vault.bin` holds only `{salt, data}` (data is ciphertext). **Forget the password = lose the data** — by design (zero-knowledge with respect to the file).

> 💡 The golden question: "What if someone gets vault.bin?" → They only have a salt + ciphertext; without the password they can't derive the key (scrypt is very slow to brute-force), so they can't decrypt.

---

## 2.7. Session & auto-lock

- On unlock, the server keeps a `Session` object with `fernet` (the key) + `salt` + `last_activity` **in RAM**.
- `is_unlocked()` returns True only when: there's a `fernet`, **not past `AUTO_LOCK_SECONDS` (300s)** since `last_activity`, and the vault file still exists.
- Every read/write calls `touch()` to push back the auto-lock deadline. `lock()` clears the key from RAM.
- This is a **module-global variable** `session` — because the app runs **locally, single-user** (no need for multi-session/login).

---

## 2.8. FastAPI & REST

- **FastAPI**: an asynchronous Python web framework, auto-validates requests with **Pydantic** (the `...Request(BaseModel)` classes).
- **REST API**: the client calls `/api/*` endpoints over HTTP (GET/POST/PATCH/DELETE), returning JSON.
- **Status codes used**: `200` OK, `400` bad request (short password, bad secret, missing CSV column), `401` vault locked / wrong password, `404` account not found.
- **StaticFiles**: FastAPI serves the static frontend (mounted at `/` last so it doesn't shadow `/api`).

---

## 2.9. PWA & Service Worker

- **PWA**: install a web app like a standalone app (own window, icon) via `manifest.json` + a service worker.
- **The service worker here**: a **network-first** strategy (always prefer the fresh version from the server, cache as offline fallback), and **never caches `/api/*`** (sensitive data + needs to be fresh). An important security decision.

---

## 2.10. Property-based testing (hypothesis) — the test highlight

- **Example-based testing**: you write a few concrete examples and assert.
- **Property-based testing**: you declare a **property that always holds**, and hypothesis **auto-generates hundreds of random inputs** to try to break it.
- The project uses hypothesis for:
  1. `seconds_remaining` always within `[1, period]` for any moment.
  2. **Round-trip** `parse_otpauth(build_otpauth(...))` preserves all parameters (randomly generated name/secret/digits/period/algorithm).
- And **RFC 6238 test vectors**: the official (time → code) table in the standard, verifying the implementation is exactly correct for SHA1/256/512.

> 💡 This is very "engineer". For algorithm code, beyond a few examples you have the **official standard** + **structured fuzzing**. Mentioning it is a big plus.

---

## 2.11. Other terms

- **Vault**: the "store" — here the file `vault.bin` holding all encrypted accounts.
- **Dynamic truncation**: the step cutting an HMAC digest into a short code (RFC 4226).
- **Master password**: one password protecting the whole vault.
- **Round-trip**: do A→B then B→A and check you get A back (e.g. export→import, build→parse).
- **`os.chmod(0o600)`**: set file permissions to owner-only read/write (security on Unix).
- **CORS**: lets a web page call an API on another origin; the project enables `allow_origins=["*"]` (safe because local).
