# 5. Frontend Deep Dive (app.js, index.html, sw.js)

> Written after directly reading `frontend/`. The frontend is **plain vanilla JS**, no framework, no build.

---

## 5.1. `index.html` — UI structure

Layout: a full-page **lock screen** + main content (4 cards) + a QR modal.

| Section | Key ids | Role |
|---------|---------|------|
| Lock screen | `#lockScreen`, `#lockPassword`, `#lockPassword2`, `#lockBtn`, `#lockHint` | Setup (2 password fields) or unlock (1 field) |
| Header | `#themeBtn`, `#toggleHideBtn`, `#lockNowBtn` | Theme / hide codes / lock now |
| Quick generate | `#quickSecret`, `#quickBtn`, `#quickCode`, `#quickCountdown` | Test any secret, no save |
| Add account | `#accName`, `#accSecret`, `#accNote`, `#otpauthUri`, scan buttons (`#scanCamBtn`/`#scanFileBtn`), `#camWrap`/`#camVideo` | Add accounts multiple ways |
| List | `#search`, `#accountList`, `#emptyMsg` | Display + search |
| Backup | `#exportBtn`, `#importBtnFile`, `#exportCsvBtn`, `#importCsvBtn` + hidden file inputs | Encrypted/CSV backup |
| QR modal | `#revealModal`, `#revealQr`, `#revealSecret`, `#revealCopy` | Re-show QR/secret |

The end of `<body>` loads `vendor/jsQR.js` **before** `app.js`.

---

## 5.2. `app.js` — Client logic (by group)

### Utilities
- **`api(path, options)`**: wraps `fetch` — auto JSON header, parse JSON, **throws `Error(data.detail)` if `!res.ok`** → callers just try/catch.
- `showError/clearError`, `formatCode` (splits "123456" → "123 456" for readability), `escapeHtml` (XSS protection when rendering names/notes).

### Quick generate
- `doQuick()`: take `quickSecret`, call `POST /generate`, show the code.
- `quickState = {secret, period}` is kept to **auto-countdown + auto-fetch a new code each period**.
- `setInterval(1000)`: computes `remaining = period - (floor(Date.now()/1000) % period)`; when `remaining === period` (new period started) → `refreshQuickCode()` fetches a new code.

### Add / import account
- `addAccount()`: `POST /accounts {name, secret, note}` → reset fields → `loadAccounts()`.
- `importUri()`: `POST /accounts/import-uri {uri}`.

### QR scanning (jsQR, in the browser)
- **From an image**: `qrFile` change → draw the image onto a hidden `canvas` → `getImageData` → `jsQR(data, w, h)` → if a string → `submitScanned`.
- **From the camera**: `getUserMedia({video:{facingMode:"environment"}})` → `requestAnimationFrame(scanCameraFrame)` scans each frame; on a valid scan → save & stop; else keep scanning. `stopCamera()` stops tracks + cancels rAF.
- `submitScanned(text)`: only accepts strings starting with `otpauth://`, sends `POST /accounts/import-uri`. **Images/frames never leave the browser** — only the otpauth string is sent.

### Lock / unlock
- `refreshStatus()`: call `GET /status` → `needSetup = !initialized`; if `locked` → `showLockScreen()`, else `hideLockScreen()` + `loadAccounts()`.
- `showLockScreen()`: if `needSetup` show 2 password fields ("Create & unlock"), else 1 field ("Unlock").
- `doUnlock()`: if setup → check ≥6 chars + 2 passwords match → `POST /setup`; else `POST /unlock`. Error → `showError`.
- `lockNowBtn` → `POST /lock` + clear list + show lock screen. `toggleHideBtn` toggles `hideCodes` (shows "••• •••").

### Account list
- `loadAccounts()`: `GET /accounts` → `accounts` → `renderAccounts()`. **If the error contains "khóa" (401 from auto-lock)** → show the lock screen.
- `renderAccounts()`: filter by `search`, build each `.account-item` (draggable): name, code (or `••• •••` if hidden), note, a **countdown ring** (`remaining` + bar `pct = remaining/period`), rename/note/reveal/delete buttons. A bad account (corrupt secret) shows `⚠ error`.
- Click the code → `copyCode()` (copies without spaces) + a "copied" effect.
- **Drag-and-drop**: `dragstart` stores `draggedId`; `drop` → `reorderByDrop(targetId)` reorders the local array + `POST /accounts/reorder {ids}` (updates the server, fire-and-forget).
- `renameAccount`/`editNote`: use `prompt()` then `PATCH`.

### Show QR / secret
- `revealAccount(id)`: `GET /accounts/{id}/reveal` → inject `qr_svg` into the modal, show the secret, copy button.

### Backup / restore
- **Encrypted export**: `prompt` for a password (≥6) → `POST /export` → download `otp-vault-backup-<date>.json`.
- **Import**: read the JSON file → `prompt` password → `POST /import` → report restored count.
- **CSV export**: `confirm` warning about plaintext secret → `fetch /export-csv` → download `.csv`.
- **CSV import**: read file → `POST /import-csv {csv}` → report `added/total_rows`.

### Loop & startup
- `setInterval(loadAccounts, 1000)`: **refresh** codes + countdown **every second** when unlocked.
- Theme: `applyTheme` toggles the `light` class on `<body>`, saves `localStorage["otpvault-theme"]`.
- `refreshStatus()` on load; registers the service worker (PWA).

> 💡 **Notable client-side security:** (1) `escapeHtml` all dynamic content → XSS protection; (2) QR images **never go to the server** (decoded by jsQR in the browser); (3) on auto-lock, the client detects the 401 and covers the screen immediately.

---

## 5.3. `sw.js` — Service Worker (PWA)

```js
const CACHE = "otp-vault-v1";
const SHELL = ["/", "/index.html", "/app.js", "/style.css", "/vendor/jsQR.js", "/manifest.json", "/icon.svg"];
// install → cache app shell + skipWaiting
// activate → delete old caches != CACHE + clients.claim
// fetch:
//   if pathname starts with "/api/" → return (NOT handled, goes straight to network, NOT cached)
//   only GET → network-first: fetch, store the new version in cache; on network error → return cache (fallback "/")
```
➡️ **The core security decision:** the service worker **never caches `/api/*`** — sensitive data (codes, account list) isn't stored in the browser cache; and **network-first** ensures always-fresh data (2FA codes change constantly).

---

## 5.4. Summary of frontend technical decisions

| Decision | Reason |
|----------|--------|
| Vanilla JS, no framework | Small single-page app; lightweight, no build |
| `api()` wraps fetch | Consistent error/JSON handling in one place |
| QR scanning via jsQR on the client | Images don't leave the device; only the otpauth string is sent |
| `setInterval(loadAccounts,1000)` | 2FA codes change constantly, need refresh + countdown |
| Detect 401 → show lock screen | Correct reaction to server auto-lock |
| SW network-first, no /api cache | Sensitive data + needs to be fresh |
| `escapeHtml` dynamic content | XSS protection |
