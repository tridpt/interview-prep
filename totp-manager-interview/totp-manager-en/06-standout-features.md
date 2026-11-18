# 6. Deep Analysis of the "Point-Scoring" Features

Each feature: the idea, how it's done, "why it's cool".

---

## 6.1. Self-implementing TOTP per RFC 6238 ⭐⭐ (the most distinctive)

**Idea:** generate 2FA codes to the same standard as Google Authenticator **without a TOTP library** — only standard `hmac`/`hashlib`/`base64`/`struct`.

**How:** decode the base32 secret → HMAC(key, counter=time//30) → **dynamic truncation (RFC 4226)** → 6-digit code. Supports SHA1/256/512, configurable digits/period.

**Why it's cool:**
- Shows **understanding of a crypto algorithm** at the implementation level, not just calling an API.
- Verified against the **official RFC 6238 test vectors** → exactly correct.
- Fewer dependencies = less attack surface.

**Possible questions:** *"Why the `at` parameter?"* → to test with a fixed time (matching the RFC table). *"What's truncation for?"* → turns a 20-byte digest into a short number, well-distributed.

---

## 6.2. The "no stored key" encryption model ⭐⭐

**Idea:** the decryption key is **never written to disk**; it exists in RAM only when unlocked.

**How:** master password → **scrypt(salt)** → 32-byte key → **Fernet (AES+HMAC)** encrypts the list. `vault.bin` holds only `{salt, ciphertext}`. `unlock` = try to decrypt (wrong password → `InvalidToken`).

**Why it's cool:**
- This is a **zero-knowledge-with-the-file** model: stealing the file is useless without the password.
- Uses the right tools: **scrypt** (brute-force-resistant KDF) + **Fernet** (authenticated encryption) — no hand-rolled crypto.
- There's a `test_vault_encrypted_at_rest` proving the raw file doesn't leak the secret.

**The golden question:** *"What if you forget the password?"* → Data is lost, **by design** (no backdoor). A deliberate security trade-off.

---

## 6.3. Auto-lock & safe session management ⭐

**Idea:** the vault auto-locks after 5 minutes of inactivity to reduce the risk of leaving the machine unlocked.

**How:** `Session.is_unlocked()` checks `last_activity`; each operation `touch()`es. When expired or **the vault file is deleted** → auto `lock()`. The client detects the 401 → shows the lock screen.

**Why it's cool:** handles edge cases too (file deleted mid-session → no 500), and the frontend reacts correctly. Both are tested.

---

## 6.4. A serious test suite: RFC vectors + property-based ⭐⭐

**Idea:** for algorithm/security code, tests must be **provably correct**.

**How:**
- **RFC 6238 test vectors**: the official (time→code) table for SHA1/256/512.
- **Property-based (hypothesis)**: `seconds_remaining` always ∈ [1, period] over 200 random inputs; `parse(build(...))` round-trip preserves every parameter.
- **Full isolation**: `conftest.py` redirects the vault to a temp dir, reduces scrypt for speed, resets the session per test.
- **3 layers**: algorithm (test_totp), vault/encryption (test_store), end-to-end API (test_api via TestClient).

**Why it's cool:** very "engineer". Saying "I test with the official standard + structured fuzzing + isolating real data" is a huge plus for a security project.

---

## 6.5. In-browser QR scanning (jsQR) ⭐

**Idea:** add accounts by scanning a QR (camera or image) — as convenient as a phone app.

**How:** draw the camera/image onto a `canvas` → `getImageData` → **jsQR** decodes → take the `otpauth://` string → send to the server to save. **The image never leaves the browser**, only the decoded string is sent.

**Why it's cool:** combines getUserMedia + Canvas + jsQR; and the decision to **decode on the client** is a security/privacy point.

---

## 6.6. "Quick generate" without unlocking ⭐

**Idea:** test any secret right away without saving or unlocking the vault.

**How:** `POST /generate {secret}` → returns code + remaining; this endpoint **isn't gated by `_ensure_unlocked`** and **doesn't touch the vault**.

**Why it's cool:** shows access-separation thinking — non-sensitive operations (pure computation) don't require unlocking.

---

## 6.7. Self-contained encrypted backup + CSV with warnings

- **Encrypted backup**: a self-contained bundle with its own `salt` + own password → restorable on another machine; **skips duplicates** (name+secret), **assigns new ids**.
- **CSV**: convenient for migration but **clearly warns** the secret is plaintext (both in the UI and README) — showing awareness of the convenience/security trade-off.

---

## 6.8. Safe PWA (network-first, no /api cache)

The service worker makes the app installable but **never caches `/api/*`** (sensitive data + constantly-changing codes) — a deliberate security decision.

---

## 6.9. "Feature ↔ technical concept" table (memorize)

| Feature | Concept you demonstrate |
|---------|-------------------------|
| Self-written TOTP | RFC 6238/4226, HMAC, base32, dynamic truncation |
| No-stored-key encryption | KDF (scrypt), Fernet/AES, salt, encryption-at-rest, zero-knowledge |
| Auto-lock/Session | session management, edge cases (file deleted), 401 |
| RFC + property-based tests | standard test vectors, hypothesis, isolated fixtures |
| Client QR scanning | getUserMedia, Canvas, jsQR, privacy |
| generate without unlock | access separation by sensitivity |
| Backup/CSV | round-trip, self-contained bundle, convenience/security trade-off |
| PWA no-/api-cache | service worker, cache security |

> 💡 **Storytelling strategy:** if asked "most proud of?", pick **self-implementing TOTP verified by test vectors** or the **no-stored-key encryption model**. Both show rare security/algorithm depth.
