# 6. Interview Question Bank & Sample Answers

70+ questions, grouped, easy to hard. Each has a suggested answer. Rephrase in your own words.

> ⭐ = very commonly asked.

---

## A. Overview & motivation

**A1. ⭐ Briefly introduce your project.**
> Quán quen is a PWA web app to save food/cafe spots by personal taste on a map — rate by your own criteria (tasty/quiet/power/date/work), suggest "where to go today", plan itineraries with real routes, discover new places from OpenStreetMap. Vanilla JS + Leaflet, no backend, data in localStorage + IndexedDB, encrypted backup, optional Gist sync. With 47 Playwright tests + CI/CD.

**A2. Why build this?**
> Google Maps doesn't remember personal "taste". I wanted a private notebook of spots by my own impressions, and a chance to practice interactive maps, advanced storage, integrating external services, and engineering discipline (tests + CI).

**A3. Who are the users?**
> People who go to cafes/restaurants often and want a private place to record spots by personal impressions — no account, no server.

**A4. ⭐ How is it different from Google Maps?**
> Google Maps shows generic community ratings; Quán quen rates by **personal, multi-criteria taste**, suggests by mood, keeps data **private** on your device, and gives you full control.

---

## B. Architecture & design

**B1. ⭐ Describe the overall architecture.**
> Client-only, no backend. index.html holds static DOM; i18n.js the dictionary; app.js the brain (state + init + render + logic); style.css the theme; sw.js offline. Maps via Leaflet. Data in localStorage (text) + IndexedDB (images). Calls Nominatim/Overpass/OSRM/Gist as needed.

**B2. ⭐ Why no backend?**
> Not needed — a personal app with private data is best kept on-device. Benefits: full privacy, simple deploy (static on GitHub Pages), no server to maintain. Cross-device sync is solved via optional GitHub Gist.

**B3. ⭐ Why vanilla JS, no framework/bundler?**
> Complex in features but state is manageable with a "render() reads state, draws UI" model. No bundler → deploy = static files, and Playwright calls global functions directly via page.evaluate. The cost is managing DOM/state myself — offset by 47 automated tests.

**B4. How does the render model work?**
> Any state change calls render(); it computes getFilteredPlaces() then redraws list, markers, heat, route, suggestions, stats. "UI is a function of state", like React but manual. Markers are reused (only add/remove/update the diff) to avoid full redraws.

**B5. app.js is 4,300 lines — downside? Fix?**
> A big file is harder to maintain. The improvement path: split into ES modules + a bundler. I keep one file now to avoid a build and to let tests call global functions easily. I note this as a limitation.

---

## C. Storage (localStorage + IndexedDB)

**C1. ⭐ Why both localStorage and IndexedDB?**
> localStorage is ~5MB and text-only; base64 images would fill it instantly. So images go in IndexedDB (large, async), text place data in localStorage. In localStorage the images field only keeps {id, category}, no dataUrl.

**C2. ⭐ How does the image flow work?**
> Startup: loadImagesFromDb() loads images into imageCache (RAM Map), hydratePlaceImages() attaches dataUrl to each place. On save: savePlaces() strips dataUrl from localStorage + persistImagesToDb() writes new images to IndexedDB. pruneOrphanImages() cleans unreferenced images.

**C3. How does IndexedDB differ from localStorage?**
> IndexedDB: async, holds objects/blobs, large capacity; localStorage: sync, text-only, ~5MB. IndexedDB is more complex (open/transaction/store) but right for large data like images.

**C4. What's the risk if browser data is cleared?**
> Everything is lost (no server). So the app has JSON/CSV/encrypted backups + Gist sync + periodic backup reminders (every 7 days).

---

## D. Maps (Leaflet)

**D1. ⭐ What is Leaflet? How do you use it?**
> An open-source interactive map library. I use tileLayer (OSM base), markers for places (custom icon + popup), markercluster to group nearby places, leaflet.heat for a density heatmap.

**D2. What are tiles? How are they cached?**
> The map is a grid of 256px image squares per zoom (z/x/y); only visible ones load. The SW caches viewed tiles (cache-first) capped at 240 (trimCache) for offline reuse without unbounded growth.

**D3. What do heatmap and markercluster solve?**
> markercluster: avoids clutter when many markers overlap (numbered cluster, click to expand). Heatmap: quickly see place density by colored regions instead of individual markers.

**D4. Why call map.invalidateSize() when toggling the sidebar?**
> Leaflet needs to know the container resized to redraw tiles correctly; otherwise the map is distorted/partially gray.

---

## E. External services & networking

**E1. ⭐ Name the external services and their roles.**
> OSM tiles (base), Nominatim (address↔coords), Overpass (find real places), OSRM (street routing), GitHub Gist (optional sync).

**E2. ⭐ What does fetchWithRetry do?**
> Calls fetch; on 429/503 it waits (900ms×attempt) then retries; on exhaustion throws an error flagged rateLimited so the UI reports "rate-limited". Improves resilience with shared services.

**E3. What if OSRM/Overpass fail or you're offline?**
> Multi-tier fallback: OSRM fails → keep the straight line; "Random nearby" fails → suggest from saved list; offline → still use saved places (just can't find new ones).

**E4. Why does the SW fetch Nominatim/Overpass/OSRM directly, no cache?**
> Dynamic, real-time data with service terms; caching would return stale results. Only tiles (rarely change) are cached.

**E5. How does discovery work?**
> discoverArea calls Overpass with `node["amenity"~"cafe|restaurant|..."]["name"](around:radius,lat,lng)`, parseOverpassPois filters valid POIs, guessTypeFromTags infers type, poiToDraft pre-fills the form for quick saving.

---

## F. Security & privacy

**F1. ⭐ How does encrypted backup work?**
> encryptBackup uses PBKDF2-SHA-256 250,000 iterations (derive a key from password + random salt) then AES-GCM (random iv). The file stores salt/iv/data base64. Restore needs the right password.

**F2. Why PBKDF2 250k iterations?**
> To slow down password brute-forcing: each guess costs 250k hash rounds. The salt defeats precomputed lookup tables (rainbow tables).

**F3. Is user data safe?**
> All data is on-device (localStorage/IndexedDB), sent nowhere unless Gist is enabled. The GitHub token is stored locally and only sent to api.github.com.

**F4. How is XSS prevented?**
> Dynamic data (including from Overpass/Nominatim) is escapeHtml/escapeAttr'd before rendering, and normalized via normalizePlace.

**F5. Why no native prompt()/confirm()?**
> For consistent UX and testability. The app has custom showPrompt/showConfirm modals; there's even a test asserting "creating a collection doesn't use a native dialog".

---

## G. Testing & CI/CD (the project's focus)

**G1. ⭐ How is the project tested?**
> 47 Playwright (E2E) tests driving a real browser, checking render/import/backup/undo/theme/i18n/filters/itinerary/OSRM/CSV/IndexedDB/Gist... Network tests mock window.fetch. Plus check-encoding.js against mojibake and node --check for syntax.

**G2. ⭐ Why can tests call app functions directly?**
> Because app.js is a classic script with everything in global scope; Playwright uses page.evaluate to run code in the page so it calls them directly (e.g. createBackupPayload, normalizePlace, getMoodScore).

**G3. How do you test network calls without a network?**
> Mock: set window.fetch = async () => ({ ok:true, json: async()=>(...) }) in the test, check the app handles it correctly. E.g. the gist push/pull test, the fetchWithRetry 429 test.

**G4. ⭐ What's your CI/CD?**
> GitHub Actions: push to main → run npm run check + npm test; only on pass does the deploy-pages job deploy GitHub Pages. So it never deploys a build broken in JS/manifest/encoding/tests.

**G5. An interesting test?**
> "images are stored in indexeddb and stripped from localStorage": create a place with an image, savePlaces + persistImagesToDb, then assert localStorage does NOT contain "data:image" while imageCache does → proving the two-tier architecture.

**G6. Why the "lucide must be pinned" test?**
> Guards against supply-chain/API-change risk: with @latest, a library change could silently break the app. The test ensures the URL has a fixed version, not @latest.

---

## H. PWA & i18n

**H1. What does your PWA cache?**
> App shell (html/css/js/icon) + CDN libs (Leaflet/markercluster/heat/lucide) + viewed tiles. 4 origin-based strategies (see E4).

**H2. What is share_target?**
> A manifest declaration so the installed PWA receives shared links from a phone (e.g. a shared Google Maps place) → auto-imports.

**H3. How does i18n work?**
> I18N_STRINGS {vi,en} in i18n.js; t(key,params) returns a string per lang. Static UI uses data-i18n + applyTranslations(); dynamic strings call t() directly. toggleLang saves to localStorage + retranslates + render.

**H4. How do users get an update?**
> Bump CACHE_VERSION; the new SW deletes old caches; an "Update" button appears when an SW is waiting, clicking it sends SKIP_WAITING to activate immediately.

---

## I. Open-ended / thinking

**I1. ⭐ Hardest challenge?**
> (Hint) Managing two-tier images smoothly (localStorage limit → IndexedDB + hydrate + orphan cleanup); or making 47 stable E2E tests with network dependencies (must mock fetch). Problem → approach → result.

**I2. ⭐ What would you improve with more time?**
> Split app.js into ES modules + a bundler; add accessibility tests with a screen reader; cache Overpass by grid tiles; optionally add a backend for real-time multi-device sync instead of manual Gist.

**I3. Where does the app break with large data / poor network?**
> localStorage ~5MB for text still has a limit (with a near-full warning); Overpass/OSRM can be slow/limited (retry + fallback). Too many markers → markercluster + heatmap reduce clutter.

**I4. The technical decision you're most proud of?**
> Engineering discipline: 47 Playwright tests + CI blocking broken deploys + encoding checks. It shows I build durable products, not just "it works".

**I5. The biggest trade-off?**
> No-backend (private, simple) at the cost of manual cross-device sync (Gist). No-bundler (easy deploy, tests call global functions) at the cost of a large app.js.

---

## J. "Specific code" questions

**J1. getFitScore vs getMoodScore?**
> getFitScore: general "fit" score (average ratings for the active taste filters + favorite/recent boosts) — shown on markers/list. getMoodScore: score for a specific suggestion mood (work/date/quiet/nearby/new...) — used for "where to go today".

**J2. Is suggestion reroll truly random?**
> No — weighted random: higher score → higher weight (score^1.25), places visited within 14 days penalized ×0.22 → variety but still sensible.

**J3. What is normalizePlace for?**
> Normalize every place from every source (manual/import/OSM/backup): fill defaults, coerce types, clamp ratings 0-5, cap tags at 12, validate dates... → data in the app is always uniform and safe.

**J4. What does parseCoordsString handle?**
> Splits a "lat, lng" string (copied from Google Maps), strips brackets/`;`/`|`, regex-captures two numbers, validates lat∈[-90,90]/lng∈[-180,180]; invalid → null.

**J5. How does getItineraryDirectionsUrl build the URL?**
> Builds a multi-stop Google Maps dir URL: origin = current location (if any) or the first stop, destination = last stop, waypoints = the middle stops joined by `|`.

**J6. What's special about savePlaces()?**
> Strips images: serializes places with images as only {id, category} (no dataUrl) before writing localStorage, then persistImagesToDb() writes images to IndexedDB, and renderDataPanel().

---

## 🎤 The 5 most common — rehearse first
1. A1 — Introduce the project.
2. C1/C2 — Why localStorage + IndexedDB & the image flow.
3. G1/G4 — Playwright tests & CI/CD.
4. F1 — Encrypted backup.
5. E2/E3 — fetchWithRetry & network fallback.

## ❓ Questions YOU should ask the interviewer
- Does the team have a CI/CD pipeline and what's the test coverage?
- Does the product use maps/geospatial, and what stack (Leaflet/Mapbox/Google)?
- How does the team handle client-side storage and offline sync?
