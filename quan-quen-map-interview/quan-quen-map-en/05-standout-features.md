# 5. Deep Analysis of the "Point-Scoring" Features

Each feature: the idea, how it's done, "why it's cool".

---

## 5.1. Two-tier storage: localStorage + IndexedDB ⭐ (frequently asked)

**Idea:** light text → localStorage; heavy images → IndexedDB.

**How:** `savePlaces()` strips `dataUrl` from places before writing localStorage (keeps only `{id, category}`); images are written to IndexedDB via `persistImagesToDb()`. On startup, `loadImagesFromDb()` → `imageCache` (RAM Map) → `hydratePlaceImages()` re-attaches dataUrl. `pruneOrphanImages()` cleans unreferenced images.

**Why it's cool:** shows understanding of the **localStorage ~5MB limit** and choosing the right store for the right data. There's a test verifying it (`images are stored in indexeddb and stripped from localStorage`). A classic question — answering it well is a big plus.

---

## 5.2. Real map service integration + network defense ⭐

**Idea:** don't host places — use OpenStreetMap (tiles + Overpass), Nominatim (addresses), OSRM (routing).

**How & defenses:**
- `fetchWithRetry`: retries on 429/503, throws a `rateLimited`-flagged error for the UI.
- **Multi-tier fallback**: OSRM fails → keep the **straight line**; "Random nearby" network error → suggest from the **saved list**; offline → still use saved places.
- The Overpass query is built dynamically by type (`discoverAmenityFilter` maps "food" → `restaurant|fast_food`).

**Why it's cool:** shows **resilience** thinking — depending on free shared services without letting the app break when they're slow/limited.

---

## 5.3. Encrypted backup with Web Crypto ⭐

**Idea:** export a safe backup file that needs a password to open.

**How:** `encryptBackup` uses **PBKDF2-SHA-256 250,000 iterations** (derive a key from the password + random salt) → **AES-GCM** with a random iv; the file stores salt/iv/data as base64 + metadata. `decryptBackup` reverses it.

**Why it's cool:** practical security knowledge (KDF against brute-force, AES-GCM encrypts + authenticates, random salt/iv each time). There's a round-trip test.

---

## 5.4. A "state → UI" render pipeline without a framework ⭐

**Idea:** every state change → call **one** `render()` that redraws the whole UI from state.

**How:** `render()` computes `getFilteredPlaces()` then calls ~12 sub-renders (list, markers, heat, route, recommendations, stats...). Markers are **reused** (diff the `markers` Map against the currently shown ids, only add/remove/update the difference) instead of redrawing from scratch.

**Why it's cool:** recreates "UI is a function of state" (like React) in vanilla — easy to reason about, fewer state bugs. Reusing markers shows performance awareness.

---

## 5.5. "Where to go today" by mood + weighted reroll ⭐

**Idea:** pick 3 places that fit the current mood; reroll for new but "smart" suggestions.

**How:** `getMoodScore(place, mood)` scores by mood (work favors power/quiet + wifi tag; date favors date score; nearby by distance; new favors **unvisited** places...). Reroll uses **weighted randomness** (`pickWeightedRecommendations`): higher score → higher weight (`score^1.25`), and **recently visited places are penalized** (×0.22) → variety while staying sensible.

**Why it's cool:** a deliberate suggestion algorithm (not pure random), balancing "fits taste" and "variety/freshness".

---

## 5.6. 47 Playwright tests + CI/CD ⭐⭐ (the biggest differentiator)

**Idea:** automatic quality assurance, never deploy a broken build.

**How:** Playwright drives a real browser to check every flow (render, import, backup, undo, theme, i18n, filters, itinerary, OSRM, CSV, IndexedDB, Gist mock...). Network tests **mock `window.fetch`**. GitHub Actions CI runs check + test, **deploys Pages only on pass**. There's also `check-encoding.js` against mojibake and a "lucide must be pinned" test.

**Why it's cool:** very rare in a personal project. Shows **mature engineering**: tests for happy paths and errors (429), mocking external services, blocking regressions. If asked "how do you ensure features don't break things?" → this is the answer.

---

## 5.7. Offline-first PWA for maps + share_target

**Idea:** install as an app, review saved places offline; receive shared links from a phone.

**How:** the SW precaches the app shell + CDN libs; **caches viewed tiles** (cache-first, capped at 240); `share_target` in the manifest receives shared Google Maps links into the app for import.

**Why it's cool:** caching map tiles is its own problem (large volume → needs `trimCache`); `share_target` is an advanced, lesser-known PWA feature.

---

## 5.8. Discovering real places from OpenStreetMap

**Idea:** not just manage saved places but **find new ones** around you.

**How:** `discoverArea`/`pickRandomNearby` call Overpass around the map center (radius 1-10km), `parseOverpassPois` filters POIs with coords + name, `guessTypeFromTags` infers the type from OSM tags, `poiToDraft` turns it into a pre-filled form draft to save quickly. Discovery markers are separate (✨).

**Why it's cool:** turns the app from a static notebook into a discovery tool, using real Overpass QL.

---

## 5.9. "Feature ↔ technical concept" table (memorize)

| Feature | Concept you demonstrate |
|---------|-------------------------|
| Two-tier storage | localStorage vs IndexedDB, 5MB limit, async storage |
| Map services + retry | REST, fetchWithRetry, 429/503 rate limits, fallback, resilience |
| Encrypted backup | Web Crypto, PBKDF2, AES-GCM, salt/iv |
| Central render() | "UI = f(state)", marker reuse, performance |
| Mood suggestions | scoring algorithm + weighted randomness |
| 47 tests + CI/CD | Playwright E2E, mock fetch, GitHub Actions, regression blocking |
| Offline PWA | service worker, tile caching, share_target |
| OSM discovery | Overpass QL, geocoding, normalizing external data |
| Leaflet | tile/marker/cluster/heat, geospatial |

> 💡 **Storytelling strategy:** if asked "what are you most proud of?", pick **47 tests + CI/CD** (technical maturity) or **two-tier storage** (solving a real limit). Both show you think beyond "it works".
