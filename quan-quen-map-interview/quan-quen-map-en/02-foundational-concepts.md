# 2. Foundational Concepts (theory)

This file explains **every term** in the project, for someone starting from scratch.

---

## 2.1. PWA & Service Worker (recap)

**PWA**: a web app installable to the home screen, running offline like native. Needs HTTPS (or localhost), a **Web App Manifest**, and a **Service Worker**.

**Service Worker (SW)**: a background script acting as a **client-side network proxy** — intercepts requests and decides cache vs network. Lifecycle: `install` (precache) → `activate` (clean old caches) → `fetch` (intercept).

**This project's specifics** (details in file 04): the SW classifies requests by **origin**:
- Nominatim / Overpass / OSRM → **fetch directly** (dynamic data, no cache).
- OpenStreetMap tiles → **cache-first** capped at 240 tiles (`trimCache`).
- Page navigation → **network-first**.
- App shell / runtime → **stale-while-revalidate**.

**`share_target`** in the manifest: when installed as a PWA, the app can **receive shared links** from a phone (e.g. share a Google Maps place into Quán quen).

---

## 2.2. Leaflet — the map library

**What is Leaflet?** An open-source JavaScript library to display **interactive maps**. It has no map data itself — it displays **tiles** (map image squares) from a provider (here, OpenStreetMap).

**Leaflet concepts in the project:**
- **Tile layer** (`L.tileLayer`): the base image layer, URL like `tile.openstreetmap.org/{z}/{x}/{y}.png` (`z`=zoom, `x/y`=tile coords).
- **Marker** (`L.marker`): a pin for a place; with a custom icon (`createMarkerIcon`) and popup (`createPopup`).
- **Layer group / markercluster**: groups markers. When many places are close, **markercluster** merges them into a numbered cluster you click to expand → avoids clutter.
- **Heat layer** (`leaflet.heat`): draws a **heatmap** of place density as colored regions instead of markers.
- **`map.setView`**, **`map.getCenter`**, **`map.invalidateSize()`** (redraw when the container resizes, e.g. toggling the sidebar).

**What's a tile (extra):** the map is divided into a grid of 256×256px image squares per zoom level; the browser only loads visible ones → light and fast. The project **caches viewed tiles** for offline reuse.

---

## 2.3. localStorage vs IndexedDB (very important)

Both store data **in the browser**, persisting after closing the tab. But they differ:

| | localStorage | IndexedDB |
|---|---|---|
| Data type | **Text only** (`JSON.stringify`) | Objects, blobs, large data |
| Capacity | ~5MB | Much larger (hundreds of MB) |
| API | Synchronous, simple | **Asynchronous** (events/promises), more complex |

**Why does the project use BOTH?** A **noteworthy architectural decision**:
- **Place images** (base64, heavy) → stored in **IndexedDB** (store `images`, key = image id, value = dataUrl).
- **Text place data** (name, coords, ratings, tags...) → stored in **localStorage**, but the `images` field **only keeps `{id, category}`, NOT the dataUrl** → so localStorage doesn't hit the 5MB limit.
- On startup: `loadImagesFromDb()` loads images into an `imageCache` (in-RAM Map), then `hydratePlaceImages()` re-attaches dataUrl to each place.
- **Orphan cleanup** (`pruneOrphanImages`): deletes IndexedDB images no place references anymore (e.g. after deleting a place).

> 💡 A great interview question: "Why move images to IndexedDB?" → because localStorage is ~5MB and base64 images would fill it instantly; IndexedDB holds much more and doesn't block the main thread (async).

---

## 2.4. Web Crypto API & encrypted backup

**Web Crypto API** (`crypto.subtle`): the browser's standard crypto API. The project uses it for **encrypted backups**:
- **PBKDF2** (Password-Based Key Derivation Function 2): turns the **user's password** into an encryption key, iterating **250,000 times** with SHA-256 + a random `salt` → makes brute-forcing the password much slower.
- **AES-GCM**: a symmetric cipher (encrypts and authenticates integrity), using a random `iv` (initialization vector).
- The encrypted file stores `salt`, `iv`, `data` (encrypted) as base64, plus metadata (`iterations`, `algorithm`). Restore needs the correct password.

**Terms:**
- **Salt**: a random string added before hashing/key derivation → same password but different salt yields a different key → defeats lookup tables (rainbow tables).
- **IV (Initialization Vector)**: a random value so the same data + same key still produces different ciphertext.

> 💡 Saying "PBKDF2 250k iterations + AES-GCM, with a random salt and iv each time" is a big plus for security knowledge.

---

## 2.5. External map services

The project doesn't host place data — it calls **free shared APIs**:
- **OpenStreetMap (OSM)**: an open map project (the "Wikipedia of maps"). Provides base tiles.
- **Nominatim**: OSM's **geocoding** service — *name/address → coords* (search) and *coords → address* (reverse). The project uses `reverseLookup` to fill the address when you tap the map.
- **Overpass API**: queries OSM data by condition. The project uses it to **find real places** around a point (`node["amenity"~"cafe|restaurant|..."]["name"](around:radius,lat,lng)`).
- **OSRM (Open Source Routing Machine)**: computes a **route** between points along streets, returning distance + time + the route geometry.
- **GitHub Gist API**: stores a "secret" JSON file on your GitHub account (needs a `gist`-scope token) → used for **backup sync** across devices.

**Side terms:** a **POI (Point of Interest)** = a place (cafe, shop...). `amenity`, `cuisine` are **tags** in OSM data describing the place type.

---

## 2.6. Async programming & network error handling

- **`async/await`**: network calls take time → async avoids freezing.
- **`fetch`**: the browser's HTTP API.
- **`fetchWithRetry(url, opts, attempts)`** (custom): if the server returns **429 (Too Many Requests)** or **503**, wait then **retry**; on exhaustion, throw an error flagged `rateLimited` so the UI can say "rate-limited". A **resilience** technique when depending on shared services.
- **`Promise.allSettled`**: waits for many promises regardless of failures (used in SW precache and writing many images).

---

## 2.7. i18n (internationalization)

- `i18n.js` defines `I18N_STRINGS = { vi: {...}, en: {...} }` (a flat dictionary keyed like `"filters.title"`).
- `t(key, params)`: returns a string for the current `lang`, falling back to `vi`, then the `key` itself. Supports replacing `{placeholder}` with `params`.
- **Static UI**: HTML attributes `data-i18n` (textContent), `data-i18n-placeholder`, `data-i18n-title`, `data-i18n-aria`; `applyTranslations()` scans and fills.
- **Dynamic strings**: JS calls `t("key")` directly (toasts, rendered buttons...).
- Switching language: `toggleLang()` saves to localStorage + `applyTranslations` + rebuilds controls + `render`.

---

## 2.8. Playwright & CI/CD (the project's differentiator)

- **Playwright**: an **end-to-end (E2E)** test framework — drives a real browser (Chromium...) like a user: opens pages, clicks, fills forms, checks the UI. The project has **47 tests**.
  - Tests call app functions directly via `page.evaluate(() => someFunction())` because everything is in **global scope** (classic script, not modules).
  - Network tests (Overpass/OSRM/Gist) **mock `window.fetch`** so no real network → fast, stable tests.
- **CI/CD**: **GitHub Actions** runs `npm run check` + `npm test` on every push to `main`; the `deploy-pages` job **runs only after tests pass** → a broken build never deploys to GitHub Pages.

---

## 2.9. Other terms

- **Haversine**: a formula for distance between two coordinates on a sphere (`haversineMeters`) — used for "near me".
- **Undo stack**: a stack of state snapshots (`pushUndoSnapshot`) to **undo** add/edit/delete/tag actions within a few seconds.
- **Mojibake**: garbled characters from wrong encoding (e.g. Vietnamese "Quán" → "QuÃ¡n"); `check-encoding.js` validates UTF-8 to prevent it.
- **Lightbox**: a zoomed image viewer with focus management + Esc to close.
- **`escapeHtml`/`escapeAttr`**: escape dynamic content (including OSM data) before inserting into HTML → prevents **XSS**.
- **`normalizePlace`/`isValidPlace`**: normalize & validate place data (fill defaults, filter valid values) — applied to every source (manual, import, OSM, backup).
