# 8. Appendix: Code Walkthrough

> ⚠️ Written after **directly reading the real code**: `app.js` (4,718 lines — read the core regions: constants/state/init/data/images/render/filter/score/CRUD/discovery/routing/backup), `sw.js`, `tests/app.spec.js`. The "most thorough" part — real functions, real variables. Open `app.js` beside you.

`app.js` is a **classic script** (no IIFE, no modules) → everything is in global scope (which is why tests can call it).

---

## PART 1 — CONSTANTS & STATE (top of file)

**Storage keys:** `STORAGE_KEY = "quan-quen-map:places:v1"`, `IMAGE_DB_NAME = "quan-quen-map-images"`, `IMAGE_STORE = "images"`, plus keys for collections/itineraries/types/theme/lang/sidebar/gist/backup-meta/discover-hint.

**Config constants:**
```js
const DEFAULT_TYPES = [ {key:"cafe",...}, {key:"food",...}, {key:"sweet",...}, {key:"drink",...} ];
let TYPES = DEFAULT_TYPES.map(t => ({...t}));   // runtime = default + custom
const PURPOSES = [taste, quiet, power, date, work];   // the 5 taste criteria
const DAY_OPTIONS = [Mon..Sun];
const PRICE_ESTIMATE_VND = { 1:40000, 2:80000, 3:150000, 4:300000 };  // spending estimate
const PHOTO_CATEGORIES = [food, space, menu, other];
const DEFAULT_CENTER = [10.7769, 106.7009];  // Ho Chi Minh City
```

**Global state (~40 vars):** `els` (DOM by id), `map`, `markers` (Map), `markerLayer`, `userLocation`, `places`, `selectedId`, `activeTypes/activePurposes/activeTags/activeAreas` (Sets), `openNowOnly`, `priceRange`, `maxDistanceKm`, `itinerary`, `collections`, `savedItineraries`, `heatLayer`, `heatmapMode`, `discoverMarkers`, `imageCache` (Map), `persistedImageIds` (Set), `lang`, `editorPhotos/editorVisits/editorOpeningDays`, `undoStack`, `pinMode`...

**`seedPlaces`:** 5 sample places — used when there's no data.

---

## PART 2 — STARTUP

```js
document.addEventListener("DOMContentLoaded", init);
function init() {
  cacheElements(); applyStoredLang(); applyStoredTheme(); loadTypes();
  setSidebarCollapsed(loadSidebarCollapsed(), {persist:false, refresh:false});
  hydrateStaticControls(); syncRangeLabels();
  places = loadPlaces(); collections = loadCollections(); savedItineraries = loadSavedItineraries();
  initMap(); bindEvents(); render(); renderDataPanel(); refreshIcons();
  setupInstallPrompt(); registerServiceWorker(); handleSharedLaunch();
  initImages();              // async
  maybeShowDiscoverHint();
}
```

- **`cacheElements()`**: an array of ~150 ids → `els[id] = document.getElementById(id)`. Centralized DOM gathering.
- **`hydrateStaticControls()`**: builds the HTML for type filters (`els.typeFilters`), taste filters (`els.purposeFilters`), the type select, rating rows (range 0-5), and day buttons — generated from `TYPES`/`PURPOSES`/`DAY_OPTIONS`, escaped via `escapeAttr`/`escapeHtml`.
- **`initMap()`**: `L.map("map",{zoomControl:false}).setView(DEFAULT_CENTER,13)`; adds zoom control bottom-right; `L.tileLayer(OSM)`; `markerLayer = L.markerClusterGroup({...})` (fallback `L.layerGroup`); `map.on("click")` → if `pinMode` or editor open, `setFormCoordinates` + `reverseLookup`.
- **`bindEvents()`**: the longest part — wires click/input/change for every control using **event delegation** (one listener per container, `event.target.closest("[data-...]")`). E.g. `els.typeFilters` click → toggle `activeTypes` → `syncFilterButtons()` + `render()`. A global `keydown` listener: Tab → `trapEditorFocus`, Escape → close modal/lightbox/pinmode/editor by priority.

---

## PART 3 — PLACE DATA

**`loadPlaces()`**: read `STORAGE_KEY`; if empty → seed 5 places (via `normalizePlace`) and save; if present → `JSON.parse` → `.map(normalizePlace).filter(isValidPlace)`; on error → notify + return `[]`.

**`savePlaces()`** (key — two-tier architecture):
```js
const serialized = places.map(p => ({ ...p, images: p.images.map(i => ({id:i.id, category:i.category})) }));
localStorage.setItem(STORAGE_KEY, JSON.stringify(serialized));   // NO dataUrl
persistImagesToDb();   // images → IndexedDB
renderDataPanel();
```

**`normalizePlace(place)`**: returns a standardized object — coerce types, `type` must be in `TYPES` (else "cafe"), `priceLevel` clamp 1-4, `ratings` clamp 0-5 per `PURPOSES`, `tags` trimmed + non-empty + max 12, `images` via `normalizeImages`, `openingHours` via `normalizeOpeningHours`, `visits` via `normalizeVisits`. **`isValidPlace`**: has `name` + finite `lat/lng`.

**`normalizeVisits`**: coerce `{id,date,rating,note}`, keep valid `YYYY-MM-DD`, sort desc by date, max 50.

---

## PART 4 — IMAGES & INDEXEDDB (distinctive)

```js
function imageDataUrl(image) { return image?.dataUrl || imageCache.get(image.id) || ""; }

function openImageDb() {            // open DB, create store "images" if missing (onupgradeneeded)
  // returns a promise cached in imageDbPromise; no indexedDB → imageDbAvailable=false
}
async function loadImagesFromDb() { // read getAllKeys + getAll → imageCache.set(key,value); persistedImageIds.add(key)
}
function writeImageToDb(id, dataUrl) { /* put(dataUrl, id) in a readwrite transaction */ }
function deleteImageFromDb(id) { /* delete(id) + remove from cache/persistedImageIds */ }

function persistImagesToDb() {      // iterate images with dataUrl, cache them, write the NOT-yet-persisted ones
  places.forEach(p => p.images.forEach(img => {
    if (!img.dataUrl) return;
    imageCache.set(img.id, img.dataUrl);
    if (!persistedImageIds.has(img.id)) writes.push(writeImageToDb(img.id, img.dataUrl));
  }));
  return Promise.allSettled(writes);
}
function pruneOrphanImages() {      // images in DB/cache no place references → delete
}
function hydratePlaceImages() {     // attach dataUrl from imageCache to each place; drop images without dataUrl
}
async function initImages() { await loadImagesFromDb(); hydratePlaceImages(); await persistImagesToDb(); savePlaces(); render(); pruneOrphanImages(); }
```
➡️ **The two-tier story**: localStorage keeps `{id,category}`; IndexedDB keeps `id→dataUrl`; `imageCache` is the RAM bridge. The most notable part for interviews.

---

## PART 5 — RENDER PIPELINE

```js
function render() {
  const filtered = getFilteredPlaces();
  els.placeCount.textContent = t("count.places", {n: places.length});
  els.filteredCount.textContent = String(filtered.length);
  renderTagFilters(); renderTagManager(); renderAreaFilters();
  renderRecommendations(); renderFavoriteReminder();
  renderList(filtered); renderMarkers(filtered); renderHeat(filtered);
  renderRoute(); renderItinerary(); renderCollections();
  renderPlaceDetail(); renderStats(); refreshIcons();
}
```

- **`renderTagFilters`/`renderAreaFilters`**: count tag/area frequency, sort, build chip buttons with `aria-pressed`.
- **`renderList(filtered)`**: builds a `.place-item` per place — thumbnail (`imageDataUrl`), `getFitScore`, badges for type/price/distance/open status/visit count/favorite, a 5-criteria mini-rating strip, tags, Edit/Favorite buttons.
- **`renderMarkers(filtered)`**: **reuses markers** — removes markers no longer in `filtered`, adds/updates the rest (`createMarkerIcon`, `createPopup`); switches markerLayer/heat per `heatmapMode`.
- **`renderHeat`**: if on, builds points `[lat,lng,weight=fitScore/100]` for `L.heatLayer`.

---

## PART 6 — FILTER, SCORING, SUGGESTIONS

**`getFilteredPlaces()`**: `places.filter(...)` applying in order: type (`activeTypes`), price (`priceRange`), distance (`maxDistanceKm` + `getDistanceFromUser`), area, collection, open-now, taste (`activePurposes` each rating ≥3), tags (all must be present), free search (normalize text + `includes`). Then `.sort()` by `els.sortMode.value` (fit/distance/recent/price/name).

**`getFitScore(place)`**:
```js
const keys = activePurposes.size ? [...activePurposes] : PURPOSES.map(p=>p.key);
const base = sum of ratings for keys / (keys.length*5);     // 0..1
return clamp(base*100 + (favorite?8:0) + recentBoost, 0, 100);
```

**`getMoodScore(place, mood)`**: switch by mood — `work` = work*16 + power*10 + quiet*7 (+10 if wifi/outlet tag); `date` = date*18 + taste*6 + quiet*5; `quiet` = quiet*20 + work*5; `taste` = taste*20 (+8 favorite); `nearby` = by distance + 0.25*fit; `new` = unvisited → 60 + 0.4*fit, else decreasing; `quick` = taste*12 + (food?18) + (price≤2?10) + quick tag. Finally: +6 if open, +5 if favorite; clamp 0-100.

**`getRecommendedPlaces`/`pickWeightedRecommendations`**: first time take top 3; reroll → **weighted random** `getRecommendationWeight = max(1,score)^1.25 * (recentlyVisited ? 0.22 : 1)` → variety yet favoring high scores, avoiding recently visited.

**`getFavoriteReminders`**: favorites where `daysSince(lastVisit) >= STALE_FAVORITE_DAYS(30)` (or never visited) → top 3.

---

## PART 7 — TAG MANAGER

**`renameTag(old, new)`**: `pushUndoSnapshot` → iterate all places renaming the tag (dedup) → if any affected, update `activeTags` + `savePlaces` + `render`; else `undoStack.pop()`. Returns the affected count. **`deleteTag`** similar (removes the tag from all places). Both have tests.

---

## PART 8 — CRUD & EDITOR

**`savePlace(event)`**: `preventDefault` → gather form values (name/type/price/coords/ratings/tags/notes/date/favorite/images `editorPhotos`/opening hours/visit history `editorVisits`) → `normalizePlace` → if invalid, notify; `pushUndoSnapshot("edit/add")`; update or `unshift` into `places`; `savePlaces()` + `closeEditor()` + `render()` + `selectPlace()` + `showUndoStatus`.

**`deleteCurrentPlace`**: `showConfirm` → `pushUndoSnapshot` → filter out of `places` → save + render + undo toast.

**`selectPlace(id, panTo)`**: set `selectedId` + render; if `panTo`, `markerLayer.zoomToShowLayer(marker, ()=>openPopup())` (de-cluster to reveal), else just open the popup.

**`toggleFavorite`/`markPlaceVisited`**: both `pushUndoSnapshot` + update + save + render + undo toast. `markPlaceVisited` blocks double-marking the same day.

**`parseCoordsString(raw)`**: strip `()`, replace `;|`→`,`, regex two numbers, validate lat∈[-90,90]/lng∈[-180,180]; invalid → null. **`handleCoordsPaste`** uses it to split into the two lat/lng fields.

**`resetFilters`**: reset every filter var to default + sync UI + render.

---

## PART 9 — DISCOVERY (Overpass) & NETWORKING

**`fetchWithRetry(url, opts, attempts=2)`**:
```js
for (i=0; i<attempts; i++) {
  res = await fetch(url, opts);
  if (res.status===429 || 503) { if attempts left: await delay(900*(i+1)); continue; else: throw {rateLimited:true}; }
  return res;
} // network error: delay(500) then retry
```

**`fetchNearbyPois(lat,lng,radius,limit)`**: build Overpass QL `[out:json][timeout:25];(node["amenity"~"${amenities}"]["name"](around:${radius},${lat},${lng}););out ${limit};`, POST via `fetchWithRetry`, `parseOverpassPois`.

**`discoverAmenityFilter(category)`**: `cafe`→"cafe", `food`→"restaurant|fast_food", `bar`→"bar|pub", `dessert`→"ice_cream|bakery", default = all.

**`discoverArea`**: get `map.getCenter()` (or userLocation), call `fetchNearbyPois` (up to 200), `showDiscoveredPlaces`. **`pickRandomNearby`**: similar but prefers unsaved POIs; network error → `fallbackRandomSaved()` (suggest from the saved list).

**`parseOverpassPois`**: filter elements with finite `lat/lon` + `tags.name`. `guessTypeFromTags`/`poiToDraft` (seen via tests): infer type from amenity, make a draft `{name,type,lat,lng}`.

---

## PART 10 — ROUTING (OSRM)

**`fetchOsrmRoute(stops, key)`**: build URL `router.project-osrm.org/route/v1/driving/{lng,lat;...}?overview=full&geometries=geojson`, `fetchWithRetry`, `parseOsrmRoute`; set `osrmRoute` then `renderRoute`/`renderItinerary`. On error → ignore (keep the straight line).

**`parseOsrmRoute(data, key)`**: take `routes[0]`, return `{key, distanceMeters, durationSec, latlngs}` (flip `[lng,lat]`→`[lat,lng]` for Leaflet).

**`formatDuration(sec)`**: <60 min → "X phút", ≥60 → "Xh MM" or "X giờ".

**`getItineraryDirectionsUrl(stops)`**: build a Google Maps `dir` URL: origin = current location (if any) or first stop, destination = last stop, waypoints = middle joined by `|`.

---

## PART 11 — ENCRYPTED BACKUP (Web Crypto)

```js
async function encryptBackup(payload, password) {
  const salt = crypto.getRandomValues(new Uint8Array(16));
  const iv   = crypto.getRandomValues(new Uint8Array(12));
  const key  = await deriveBackupKey(password, salt);     // PBKDF2-SHA-256, 250000 iters
  const cipher = await crypto.subtle.encrypt({name:"AES-GCM", iv}, key, encode(JSON));
  return { app, version:2, encrypted:true, algorithm:"AES-GCM", kdf:"PBKDF2-SHA-256",
           iterations:250000, salt(b64), iv(b64), data(b64) };
}
async function deriveBackupKey(password, salt, iterations=250000) {
  importKey(password, "PBKDF2") → deriveKey({name:"PBKDF2", salt, iterations, hash:"SHA-256"}, → AES-GCM 256)
}
```
`decryptBackup` reverses it (reads salt/iv/iterations from the file). There's a round-trip test.

---

## PART 12 — SERVICE WORKER (sw.js)

```
CACHE_VERSION = "taste-map-v36"
APP_SHELL = [local files + CDN Leaflet/markercluster/heat/lucide]
install  → cache.add each asset (Promise.allSettled) → skipWaiting
activate → delete caches != CACHE_VERSION → clients.claim
message  → SKIP_WAITING → skipWaiting
fetch (GET):
  origin Nominatim/OSRM/Overpass → fetch directly
  tile.openstreetmap.org → cacheFirst(TILE_CACHE, 240)  // trimCache recursively deletes oldest
  navigate → networkFirstPage (offline → "./" or index.html)
  app shell → staleWhileRevalidate(APP_SHELL_CACHE)
  else → staleWhileRevalidate(RUNTIME_CACHE)
```

---

## PART 13 — TESTS (tests/app.spec.js) — 47 cases

Call app functions via `page.evaluate`. Test groups: render & UI, Google Maps import, encrypted backup round-trip, undo, PWA manifest, open-now, stats, share payload, random nearby, parse Overpass POI, price/area filters, itinerary + OSRM parse, CSV, collections, heatmap, "new" mood, favorite reminder, save/load itinerary, rename/delete tag, dialog accessibility (focus trap + Esc), photo albums, i18n static+dynamic, **IndexedDB images + localStorage strip**, rehydrate after reload, orphan image prune, lightbox, discover amenity filter, explore area (mock Overpass 25 markers), coords parse, **lucide pinned version**, **fetchWithRetry 429**, no-native-prompt modal, gist push/pull (mock).

---

## 🎯 Quick "function ↔ purpose" reference

| Group | Main functions |
|------|----------------|
| Startup | `init`, `cacheElements`, `hydrateStaticControls`, `initMap`, `bindEvents` |
| Data | `loadPlaces`, `savePlaces`, `normalizePlace`, `isValidPlace`, `normalizeVisits/OpeningHours` |
| Images/IndexedDB | `openImageDb`, `loadImagesFromDb`, `writeImageToDb`, `persistImagesToDb`, `hydratePlaceImages`, `pruneOrphanImages`, `imageDataUrl`, `initImages` |
| Render | `render`, `renderList/Markers/Heat/Route/Itinerary/Recommendations/Stats/TagFilters/AreaFilters/FavoriteReminder/DataPanel` |
| Filter/score | `getFilteredPlaces`, `getFitScore`, `getMoodScore`, `getRecommendedPlaces`, `pickWeightedRecommendations`, `getFavoriteReminders`, `getOpeningStatus(At)` |
| Tags | `renameTag`, `deleteTag` |
| CRUD | `openEditor`, `savePlace`, `deleteCurrentPlace`, `selectPlace`, `toggleFavorite`, `markPlaceVisited`, `parseCoordsString`, `resetFilters` |
| Discovery | `fetchNearbyPois`, `discoverArea`, `pickRandomNearby`, `parseOverpassPois`, `discoverAmenityFilter`, `guessTypeFromTags`, `poiToDraft` |
| Network | `fetchWithRetry`, `delayMs` |
| Routing | `fetchOsrmRoute`, `parseOsrmRoute`, `formatDuration`, `getItineraryDirectionsUrl` |
| Backup | `encryptBackup`, `decryptBackup`, `deriveBackupKey`, `createBackupPayload`, `importCsvText`, `pushGistBackup`, `pullGistBackup` |
| i18n/theme | `t`, `applyTranslations`, `toggleLang`, `applyTheme`, `toggleTheme` |
| Utilities | `escapeHtml/escapeAttr`, `haversineMeters`, `showStatus/showUndoStatus/showConfirm/showPrompt` |

> 💡 Master this table + read file 08 once, and you can **point at any function in app.js and explain it**.
