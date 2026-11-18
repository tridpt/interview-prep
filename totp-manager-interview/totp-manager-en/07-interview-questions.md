# 7. Interview Question Bank & Sample Answers

70+ questions, grouped, easy to hard. Each has a suggested answer. Rephrase in your own words.

> ⭐ = very commonly asked.

---

## A. Overview & motivation

**A1. ⭐ Briefly introduce your project.**
> OTP Vault is a locally-run 2FA (TOTP) manager — paste a secret, get a 6-digit code that changes every 30s per RFC 6238. It stores many accounts in an encrypted vault protected by a master password (scrypt + Fernet/AES), the key only in RAM, auto-locking after 5 minutes. The FastAPI backend self-implements TOTP from standard libraries; the vanilla-JS frontend (PWA) has QR scanning. The test suite includes RFC 6238 test vectors and property-based testing.

**A2. Why build this?**
> I wanted a secure place on a computer to manage many 2FA codes, and a chance to practice real-world crypto: implementing TOTP correctly, encryption-at-rest with a KDF, and serious testing.

**A3. Who are the users?**
> Individuals managing their own 2FA codes on a computer, secured by a master password.

**A4. ⭐ What's it for, any abuse risk?**
> For your own accounts or ones you're allowed to manage. The README states it's NOT a mass fake-account farming tool. The app is local-only, single-user.

---

## B. TOTP & the algorithm

**B1. ⭐ How does TOTP work? Explain step by step.**
> Decode the base32 secret into a key → counter = floor(time/30) → HMAC(key, counter) gives a digest → dynamic truncation (offset from the last 4 bits, read 4 bytes, drop the sign bit) → number % 10^digits, zero-pad → 6-digit code. Server and client compute the same code from secret + time.

**B2. ⭐ How is TOTP different from HOTP?**
> HOTP (RFC 4226) uses an incrementing counter; TOTP (RFC 6238) uses counter = time/period. TOTP is HOTP with a time-based counter.

**B3. Why is the secret base32?**
> Base32 uses only A-Z, 2-7 — easy to type, less character confusion (no 0/O, 1/l). It's the authenticator standard.

**B4. ⭐ Why self-write TOTP instead of pyotp?**
> TOTP is just HMAC + truncation per the RFC — self-writing with standard hmac/hashlib reduces dependencies, gives full understanding, and I verified it against the official RFC 6238 test vectors. The hard crypto (vault encryption) I do with the standard cryptography library, not hand-rolled.

**B5. What is dynamic truncation, and why?**
> The RFC 4226 step turning an HMAC digest (20 bytes) into a short number: take the last 4 bits as offset, read 4 bytes at that offset, drop the sign bit. Purpose: a short, typeable code that's still well-distributed and hard to guess.

**B6. What does normalize_secret do?**
> Strip spaces/dashes, uppercase, add '=' padding to a multiple of 8, then b32decode to validate. Idempotent. Because users copy messy secrets.

**B7. Can seconds_remaining return 0?**
> No — always within [1, period]. There's a property-based test over 200 random times.

**B8. How many algorithms are supported?**
> SHA1 (default), SHA256, SHA512 — via the `_ALGOS` dict. A bad algorithm → TOTPError.

---

## C. Encryption & security (the focus)

**C1. ⭐ How is data encrypted?**
> Master password → scrypt(salt) → 32-byte key → Fernet (AES-128-CBC + HMAC-SHA256) encrypts the JSON list. vault.bin holds only {salt, ciphertext}.

**C2. ⭐ What is scrypt, why use it?**
> A KDF turning a password into a key, intentionally memory + CPU heavy (N=32768, r=8, p=1) to resist brute force. A random 16-byte salt defeats rainbow tables.

**C3. ⭐ What is Fernet?**
> A safe symmetric-encryption standard from the cryptography library: AES-128-CBC + HMAC-SHA256, encrypts and authenticates integrity. Key = base64(32 bytes from scrypt).

**C4. ⭐ What does "no stored key" mean?**
> The Fernet key exists only in RAM (Session.fernet) when unlocked. The file has only salt + ciphertext. Forget the password = lose the data, no backdoor.

**C5. ⭐ How does unlock authenticate without storing a hash?**
> unlock() derives the key from the password then tries fernet.decrypt(data). Successful decryption = correct password; InvalidToken = wrong (raise WrongPassword → 401). Decrypting successfully IS the proof.

**C6. Is getting vault.bin dangerous?**
> Not if the password is strong: an attacker only has salt + ciphertext and must brute-force the password through scrypt (very slow). A test proves the file has no plaintext secret.

**C7. What's the risk with CSV?**
> CSV contains the secret in PLAINTEXT (unencrypted). The app warns clearly in the UI (confirm) and README, advising to delete the file after use. A deliberate convenience trade-off.

**C8. How does the encrypted backup differ from the vault file?**
> The backup has its own salt + its own password (set at export time) so it's self-contained, restorable on another machine. The vault uses the current machine's master password.

---

## D. Session & auto-lock

**D1. ⭐ How does auto-lock work?**
> Session stores last_activity; is_unlocked() returns False if past AUTO_LOCK_SECONDS (300s). Each operation calls touch() to push the deadline. Expired → lock → API returns 401 → frontend shows the lock screen.

**D2. What if the vault file is deleted mid-session?**
> is_unlocked() checks the file exists; if gone → auto lock() and return locked, avoiding a 500 from reading a missing file. There's a test for this.

**D3. Why a single global Session?**
> The app runs locally, single-user — no need for multi-session/login. One shared module variable `session` is enough and simple.

**D4. How does the frontend know about auto-lock?**
> It calls /accounts every second; if the server returns 401 (error containing "khóa"), the client shows the lock screen.

---

## E. API & FastAPI

**E1. Which endpoints don't need unlock? Why?**
> /status, /setup, /unlock, /lock, /generate. generate only computes a code from the passed secret, doesn't touch the vault, so it's safe to leave open.

**E2. Status code rules?**
> 200 OK; 400 bad request (password <6, bad secret, missing CSV column); 401 vault locked / wrong password; 404 account not found.

**E3. How does _code_payload handle errors?**
> It catches TOTPError while computing a code and returns {code:None, error}; so one bad-secret account doesn't crash the whole list.

**E4. What is Pydantic for?**
> It defines request schemas (AddAccountRequest...); FastAPI auto-validates JSON, returning 422 on type errors.

**E5. How is the frontend served?**
> main.py mounts StaticFiles at "/" (placed last so it doesn't shadow /api), with html=True to serve index.html.

---

## F. Frontend & PWA

**F1. ⭐ How does QR scanning work? Is the image sent to the server?**
> No. Draw the camera/image onto a canvas → getImageData → jsQR decodes in the browser → only the otpauth:// string is sent to the server. The image never leaves the device.

**F2. How does the list keep codes up to date?**
> setInterval(loadAccounts, 1000): every second it calls /accounts for fresh codes + remaining, re-rendering the countdown ring.

**F3. ⭐ Does the service worker cache /api? Why?**
> Never. Sensitive data (codes, list) + needs to be fresh (codes change constantly). The SW uses network-first for the app shell and skips /api entirely.

**F4. XSS protection on the client?**
> escapeHtml all dynamic content (names, notes, errors) before inserting into innerHTML.

**F5. How is drag-to-reorder saved?**
> dragstart stores draggedId; drop → reorderByDrop reorders the local array + POST /accounts/reorder {ids} so the server saves it (fire-and-forget).

---

## G. Testing (the highlight)

**G1. ⭐ How do you test the project?**
> 3 pytest layers: test_totp (algorithm), test_store (vault/encryption), test_api (end-to-end via TestClient). With RFC 6238 test vectors and property-based testing via hypothesis.

**G2. ⭐ What are RFC 6238 test vectors?**
> The official (algorithm, time → 8-digit code) table in the standard. I assert generate_code matches exactly for SHA1/256/512 at moments 59, 1111111109... → proving the implementation is exactly correct.

**G3. ⭐ What is property-based testing? Where do you use it?**
> Declare a property that always holds; hypothesis auto-generates hundreds of random inputs to break it. I use it for seconds_remaining (always ∈[1,period]) and the round-trip parse(build(...)) preserving parameters.

**G4. ⭐ Do tests touch real data?**
> No. conftest.py has an autouse fixture redirecting VAULT_PATH to a temp dir, reducing scrypt for speed, resetting the session each test. Never writes to the real vault.

**G5. A notable security test?**
> test_vault_encrypted_at_rest: add an account then read the raw file, assert the secret + name do NOT appear in plaintext → proving encryption-at-rest.

**G6. What does "Req X.Y" in tests mean?**
> The project is spec-driven (.kiro/specs): each test maps to a specific requirement in the spec, ensuring full requirement coverage.

---

## H. Open-ended / thinking

**H1. ⭐ Hardest challenge?**
> (Hint) Implementing TOTP bit-exact to match the RFC vectors (endianness, truncation, base32 padding); or designing the no-stored-key encryption model that still authenticates the password. Problem → approach → result.

**H2. ⭐ What would you improve with more time?**
> Rate-limit password attempts; raise scrypt N per hardware; encrypt each account separately instead of the whole vault (less exposure on decrypt); support multi-user if deployed for a team; time-skew tolerance for clock drift.

**H3. Known weaknesses/limitations?**
> Local-only, single-user (no multi-user). CSV leaves the secret plaintext (warned). The key in RAM means anyone who can read the process memory could theoretically see it while unlocked. Forget the password = lose the data.

**H4. The technical decision you're most proud of?**
> Self-implementing TOTP correctly per the RFC and verifying with official test vectors + property-based — showing I understand the crypto algorithm at the implementation level, not just gluing a library.

**H5. The biggest trade-off?**
> "No stored key" → high security but forgetting the password means data loss (no backdoor). Local-only → safe but no remote access (put it behind a VPN if needed).

**H6. What if the machine's clock is off?**
> TOTP depends on time; being off by minutes yields wrong codes. Improvement: allow a ±1 period window when validating (real servers do this); this app only generates codes so it relies on the machine clock.

---

## I. "Specific code" questions

**I1. What does struct.pack(">Q", counter) do?**
> Packs the counter into 8 bytes unsigned big-endian — the message format for HMAC per the RFC.

**I2. What's & 0x7FFFFFFF for?**
> Drops the highest (sign) bit of the 32-bit number read from the digest, ensuring it's positive — a standard step of dynamic truncation.

**I3. Why os.chmod(0o600) when writing the vault?**
> Set owner-only read/write permissions (on Unix) → other users on the machine can't read the vault.

**I4. What does import_backup dedupe on?**
> The (name, secret) pair. Existing entries are skipped; new entries get new ids (uuid). Avoids duplication on restore.

**I5. How does reorder_accounts handle junk/missing ids?**
> Nonexistent ids are skipped; unmentioned accounts are kept at the end in their old order → never loses data.

**I6. Why add a BOM (\ufeff) to the CSV?**
> So Excel recognizes UTF-8 and displays Vietnamese without font issues.

---

## 🎤 The 5 most common — rehearse first
1. A1 — Introduce the project.
2. B1/B4 — TOTP step by step & why self-written.
3. C1/C4/C5 — Encryption, "no stored key", unlock authentication.
4. G2/G3 — RFC test vectors + property-based.
5. F1/F3 — Client QR scanning & SW not caching /api.

## ❓ Questions YOU should ask the interviewer
- How does the team handle sensitive/crypto data (KDFs, encryption-at-rest)?
- What's the testing process for security code (test vectors, property-based, fuzzing)?
- Do you use spec-driven / formal requirements before coding?
