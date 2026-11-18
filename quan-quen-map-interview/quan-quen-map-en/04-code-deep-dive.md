# 4. Code Deep Dive (file by file)

Read alongside the real code in `quan-quen-map/`. For a function-by-function walk of app.js, see `08-code-walkthrough.md`.

---

## 4.1. `index.html` — Skeleton & static DOM (~574 lines)

**Job:** holds all static DOM — toolbar, sidebar (filters, suggestions, itinerary, data), the `#map` area, list + detail panels, editor (add/edit-place form as a dialog), lightbox, modal.

**Key points:**
- `<head>`: declares PWA (`manifest`, `theme-color`, apple metas) + loads Leaflet/markercluster CSS.
- End of `<body>`: loads CDN libs (Leaflet, markercluster, heat, Lucide) then **`i18n.js` → `app.js`** (mandatory order).
- All translatable text uses `data-i18n` / `data-i18n-placeholder` / `data-i18n-title` / `data-i18n-aria`.
- The **editor** (`#editorPanel`) is `role="dialog" aria-modal="true"` — with focus management (focus trap).

---

## 4.2. `i18n.js` — Translation dictionary (~688 lines)

- Defines `I18N_STRINGS = { vi: {...}, en: {...} }` — a flat dictionary keyed (`"filters.title"`, `"ui.edit"`, `"msg.placeSaved"`...).
- Mostly **data** (key→string pairs for both languages). The translation logic (`t`, `applyTranslations`) lives in `app.js`.
- Loaded **before** app.js so `I18N_STRINGS` is ready.

---

## 4.3. `app.js` — The brain (~4,718 lines)

A **classic script** (no IIFE, no modules) → all functions in global scope. Groups:

### Constants & state (top of file)
- Storage keys (`STORAGE_KEY`, `IMAGE_DB_NAME`...), config constants (`DEFAULT_TYPES`, `PURPOSES`, `DAY_OPTIONS`, `PHOTO_CATEGORIES`, `PRICE_ESTIMATE_VND`, `RANDOM_NEARBY_RADIUS`...), `seedPlaces` (5 samples).
- ~40 global state variables (see file 03 §3.6).

### Startup & DOM
- `init()` (see file 03), `cacheElements()` (gather ~150 ids into `els`), `hydrateStaticControls()` (build filter/select/rating controls from `TYPES`/`PURPOSES`/`DAY_OPTIONS`), `bindEvents()` (wire all events — the longest part), `initMap()` (Leaflet + tiles + markercluster + map-click to pin coords).

### Storage & data
- `loadPlaces()` (read localStorage, seed if empty), `savePlaces()` (strip images from localStorage + `persistImagesToDb()`), `normalizePlace()` / `isValidPlace()`, `normalizeVisits()`, `normalizeOpeningHours()`.

### Images & IndexedDB
- `openImageDb()` (open DB, create `images` store), `loadImagesFromDb()` (load into `imageCache`), `writeImageToDb`/`deleteImageFromDb`, `persistImagesToDb()` (write new images), `hydratePlaceImages()` (attach dataUrl from cache), `pruneOrphanImages()` (delete unreferenced images), `imageDataUrl()`.

### Render (synchronous)
- `render()` orchestrates; sub-renders: `renderList`, `renderMarkers`, `renderHeat`, `renderRoute`, `renderItinerary`, `renderCollections`, `renderPlaceDetail`, `renderStats`, `renderTagFilters`, `renderAreaFilters`, `renderRecommendations`, `renderFavoriteReminder`, `renderDataPanel`.

### Filter, scoring, suggestions
- `getFilteredPlaces()` (apply all filters + 5 sort modes), `getFitScore()` (fit score on markers/list), `getMoodScore(place, mood)` (mood-based score), `getRecommendedPlaces`/`pickWeightedRecommendations` (pick 3, **weighted reroll** avoiding recently visited), `getFavoriteReminders()`.
- Opening status: `getOpeningStatus()` / `getOpeningStatusAt(place, day, minutes)`. Area: `getPlaceArea()`/`parseArea()`.

### CRUD & editor
- `openEditor`, `closeEditor`, `savePlace`, `deleteCurrentPlace`, `selectPlace`, `toggleFavorite`, `markPlaceVisited`, `setFormCoordinates`, `handleCoordsPaste`/`parseCoordsString`. With **undo** via `pushUndoSnapshot` + `showUndoStatus`.

### Map & discovery
- `createMarkerIcon`, `createPopup`, `locateUser`, `toggleHeatmap`, `pickRandomNearby`, `discoverArea`, `fetchNearbyPois` (Overpass), `parseOverpassPois`, `guessTypeFromTags`, `poiToDraft`, `showDiscoveredPlaces`, `discoverAmenityFilter`.

### Routing
- `fetchOsrmRoute`/`parseOsrmRoute` (OSRM), `renderRoute`, `getItineraryDirectionsUrl` (multi-stop Google Maps URL), `openExternalMap`/`getGoogleMapsUrl`/`getAppleMapsUrl`, `formatDuration`.

### Data (backup/import)
- `createBackupPayload`, `exportPlainBackup`, `exportEncryptedBackup`, `encryptBackup`/`decryptBackup`/`deriveBackupKey` (Web Crypto), `importPlaces`, `importCsvText`/`parseCsv`/`exportCsv`, `saveGistSettings`/`pushGistBackup`/`pullGistBackup`, `importFromGoogleMaps`/URL parsing.

### i18n, theme, utilities
- `t`, `applyTranslations`, `toggleLang`, `applyTheme`, `toggleTheme`, `typeLabel`, `purposeLabel`; `escapeHtml`/`escapeAttr`, `haversineMeters`, `fetchWithRetry`, `showStatus`/`showUndoStatus`/`showConfirm`/`showPrompt` (in-app modal/toast, **no native `prompt()`**).

---

## 4.4. `sw.js` — Service Worker (~148 lines)

The 4 strategies are in file 03 §3.7. Functions: `networkFirstPage`, `staleWhileRevalidate`, `cacheFirst`, `trimCache` (caps tile count), `isAppShellRequest`, `isCacheable`. `CACHE_VERSION = "taste-map-v36"`.

> 💡 **Common question:** "Why fetch Nominatim/Overpass/OSRM directly, no cache?" → dynamic/time-sensitive data + service terms; caching would return stale results and waste space. Only **tiles** (rarely change) are cached for offline reuse.

---

## 4.5. `tests/app.spec.js` — 47 Playwright tests (~774 lines)

This is the **project's highlight**. Real test examples:
- `app render` — checks the 5 seed places render, 3 suggestions, tag filtering, sidebar collapse.
- `encrypted backup can decrypt to original payload` — encrypt then decrypt, compare.
- `undo restores data after adding a place` — add → click Undo → back to 5.
- `images are stored in indexeddb and stripped from localStorage` — verifies the **two-tier** architecture.
- `fetchWithRetry retries on 429 then surfaces a rate-limit error` — mocks 429, checks retry + the `rateLimited` flag.
- `gist push/pull uses mocked fetch` — mocks the GitHub API.
- `lucide is pinned to a fixed version (not latest)` — guards against surprise library changes.
- Plus tests for i18n, theme, price/area/open filters, itinerary + OSRM, CSV, collections, heatmap, lightbox, dialog accessibility, Overpass discovery...

**How tests call functions:** `page.evaluate(() => someAppFunction())` runs code **in the page** → calls the app's global functions directly. Network tests do `window.fetch = async () => ({...})` to mock.

---

## 4.6. `.github/workflows/ci.yml` — CI/CD

Push to `main` → job `checks` (`npm run check` + `npm test`) → if it passes, job `deploy-pages` deploys GitHub Pages. **No pass, no deploy** → never ships a broken build.

---

## 4.7. Summary of technical decisions

| Decision | Reason |
|----------|--------|
| Vanilla JS, no bundler | Deploy = static files; tests call global functions; lightweight |
| localStorage + IndexedDB | Text fast/simple; large images don't fill the 5MB localStorage |
| Central `render()` (state→UI) | Consistent, easy to reason about, no framework |
| 4 origin-based cache strategies | Dynamic data fetched directly; tiles cache-first; pages network-first |
| `fetchWithRetry` + fallbacks | Resilient to shared services (429/503); OSRM fails → straight line |
| AES-GCM + PBKDF2 250k backup | Protects data when sharing/syncing |
| Custom modal/toast (no `prompt()`) | Consistent UX + testable (a test blocks native prompt) |
| 47 Playwright tests + CI | Keep quality, block broken deploys |
| escapeHtml all data (incl. OSM) | Prevent XSS |
| Pinned CDN versions | Avoid surprise library changes breaking the app |
