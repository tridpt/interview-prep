# 1. Project Overview — OTP Vault (totp-manager)

## 1.1. What is OTP Vault?

OTP Vault is a locally-run **2FA (two-factor authentication) code manager**. You paste an account's **secret key** (obtained when enabling 2FA), and the app generates a **6-digit code that changes every 30 seconds** — exactly like Google Authenticator or Authy. The difference: it stores **multiple accounts**, shows all codes at once, and protects them with a **master password + local encryption**.

### Core points to remember
- Generates codes per the **TOTP (RFC 6238)** standard — the common standard of all authenticator apps.
- **Master password**: the encryption key is derived from the password via **scrypt**, **never written to disk**. Forget it = lose the data.
- **Auto-lock after 5 minutes** of inactivity + manual lock + hide/show codes.
- Runs **local-only** (`127.0.0.1`), no multi-user, not exposed to the Internet.
- Backend **FastAPI (Python)**, frontend **vanilla JS (PWA)**, with a **rigorous test suite** (RFC vectors + property-based).

> ⚠️ The README states clearly: use it for **your own** accounts or those you're allowed to manage; it's not a tool for mass-farming fake accounts.

## 1.2. What problem does it solve?

1. **Managing many 2FA codes in one secure place on a computer** — not just on a phone.
2. **Protecting sensitive data**: 2FA secret keys are extremely sensitive; the app encrypts everything with the master password and sends nothing anywhere.
3. **Migration/backup**: export an encrypted vault, re-show QR/secret to add to a phone, import/export CSV.
4. **Demonstrating technical ability**: implementing a crypto algorithm (TOTP from HMAC) yourself, encryption-at-rest with a KDF, session/auto-lock management, and **serious testing** (standard test vectors + property-based) — the biggest point-scorer.

> 💡 **Interview tip:** this is a **security + crypto algorithm** project — very different from typical CRUD apps. Emphasize: implementing TOTP correctly per the RFC from standard libraries, the "no stored key" encryption model, and a test suite with **RFC 6238 test vectors** + **property-based testing**. These show real depth, not just gluing APIs.

## 1.3. Main features

| Group | Features |
|-------|----------|
| **Code generation** | Quick generate from a secret (no unlock, no save); show code + countdown for all accounts |
| **Security** | Master password (scrypt), encrypted vault (Fernet/AES), auto-lock 5 min, manual lock, hide/show codes |
| **Account management** | Add (name + secret / paste `otpauth://` / **scan QR** camera or image), rename, note, drag-to-reorder, delete |
| **Re-show** | Show QR (SVG) + secret of each account to add it to another app |
| **Backup** | Export **encrypted** bundle (separate password), restore (skip duplicates); import/export **CSV** (warns secret is plaintext) |
| **UI** | Light/dark (localStorage), search, click code to copy, **installable PWA** |

## 1.4. Tech stack

**Backend (Python):**
- **FastAPI** `0.115.0` — web framework, defines the REST API.
- **uvicorn[standard]** `0.30.6` — ASGI server.
- **cryptography** `>=43` — `Fernet` (symmetric encryption) + `Scrypt` (KDF). **This is where encryption happens.**
- **segno** `>=1.6` — generate QR codes (SVG) in pure Python for "re-show QR".
- **TOTP self-written** with `hmac`, `hashlib`, `base64`, `struct` from the standard library — **no external library**.

**Frontend:**
- **Plain HTML/CSS/JS (vanilla)** — no framework, no build.
- **jsQR** (vendored in the repo) — decode QR in the browser (camera/image).
- Web APIs: Fetch, getUserMedia (camera), Canvas, Clipboard, localStorage, Service Worker (PWA).

**Testing:**
- **pytest** + **hypothesis** (property-based testing) + **httpx** (for `fastapi.testclient.TestClient`).

> 💡 **A trap question: "Why self-write TOTP instead of using `pyotp`?"** A good answer: "TOTP is just HMAC + dynamic truncation per RFC 6238 — self-writing it with standard `hmac`/`hashlib` reduces dependencies (less attack surface), gives full understanding of the algorithm, and I verified it against the **official RFC 6238 test vectors** (SHA1/256/512). The genuinely hard crypto (vault encryption) I do with the standard `cryptography` library, not hand-rolled."

## 1.5. How to run

```bat
cd /d d:\AI_App\totp-manager
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt

cd backend
uvicorn main:app --reload --port 8200
```
Open `http://127.0.0.1:8200` — the first time it asks you to **create a master password**.

Run tests:
```bat
pip install -r requirements-dev.txt
pytest
```

> 🔒 The app **only listens on `127.0.0.1`** → only this machine can access it. Data is encrypted in `storage/vault.bin`, and `storage/` is gitignored.

## 1.6. Folder structure

```
totp-manager/
├── backend/
│   ├── config.py   # Vault path, scrypt params, auto-lock time
│   ├── totp.py     # TOTP generation (RFC 6238), normalize secret, parse/build otpauth
│   ├── store.py    # Encrypted vault (Session): setup/unlock/lock, CRUD, export/import
│   └── main.py     # FastAPI: /api/* endpoints, serves the static frontend
├── frontend/
│   ├── index.html  # UI: lock screen, quick generate, list, backup, QR modal
│   ├── app.js      # All client logic (API calls, render, QR scan, drag, theme)
│   ├── style.css   # Light/dark UI
│   ├── sw.js       # Service worker (PWA, network-first, NEVER caches /api)
│   ├── manifest.json
│   └── vendor/jsQR.js  # QR decoding library (client)
├── tests/          # test_totp.py, test_store.py, test_api.py
├── conftest.py     # Fixture: isolated temp vault + reduced scrypt for fast tests
├── storage/vault.bin   # Encrypted vault (gitignored)
├── requirements.txt / requirements-dev.txt
└── .kiro/specs/    # Requirement specs (spec-driven: tests tagged "Req X.Y")
```

> 💡 **Worth noting:** the project has a `.kiro/specs/` folder — developed **spec-driven** (requirements first). Tests are annotated "Req 2.1", "Req 6.3"... meaning each test maps to a specific requirement. Mentioning this shows a disciplined workflow.
