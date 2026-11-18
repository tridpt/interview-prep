# 8. Phụ lục: Đi bộ qua code (Code Walkthrough)

> ⚠️ Viết sau khi **đọc trực tiếp code thật**: `app.js` (4718 dòng — đã đọc các vùng cốt lõi: hằng/state/init/dữ liệu/ảnh/render/lọc/điểm/CRUD/khám phá/định tuyến/backup), `sw.js`, `tests/app.spec.js`. Đây là phần "cặn kẽ nhất" — đi qua hàm thật, biến thật. Mở `app.js` bên cạnh để đối chiếu.

`app.js` là **classic script** (không IIFE, không module) → mọi hàm ở global scope (nên test gọi được).

---

## PHẦN 1 — HẰNG SỐ & STATE (đầu file)

**Khóa lưu trữ:** `STORAGE_KEY = "quan-quen-map:places:v1"`, `IMAGE_DB_NAME = "quan-quen-map-images"`, `IMAGE_STORE = "images"`, và các khóa cho collections/itineraries/types/theme/lang/sidebar/gist/backup-meta/discover-hint.

**Hằng cấu hình:**
```js
const DEFAULT_TYPES = [ {key:"cafe",...}, {key:"food",...}, {key:"sweet",...}, {key:"drink",...} ];
let TYPES = DEFAULT_TYPES.map(t => ({...t}));   // runtime = mặc định + tùy chỉnh
const PURPOSES = [taste, quiet, power, date, work];   // 5 tiêu chí gu
const DAY_OPTIONS = [T2..CN];
const PRICE_ESTIMATE_VND = { 1:40000, 2:80000, 3:150000, 4:300000 };  // ước tính chi tiêu
const PHOTO_CATEGORIES = [food, space, menu, other];
const DEFAULT_CENTER = [10.7769, 106.7009];  // TP.HCM
```

**State toàn cục (~40 biến):** `els` (DOM theo id), `map`, `markers` (Map), `markerLayer`, `userLocation`, `places`, `selectedId`, `activeTypes/activePurposes/activeTags/activeAreas` (Set), `openNowOnly`, `priceRange`, `maxDistanceKm`, `itinerary`, `collections`, `savedItineraries`, `heatLayer`, `heatmapMode`, `discoverMarkers`, `imageCache` (Map), `persistedImageIds` (Set), `lang`, `editorPhotos/editorVisits/editorOpeningDays`, `undoStack`, `pinMode`...

**`seedPlaces`:** 5 quán mẫu (cafe ban công, mì vịt hẻm, bánh ngọt, trà rooftop, cơm trưa) — dùng khi chưa có dữ liệu.

---

## PHẦN 2 — KHỞI ĐỘNG

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

- **`cacheElements()`**: một mảng ~150 id → `els[id] = document.getElementById(id)`. Cách gom DOM tập trung.
- **`hydrateStaticControls()`**: dựng HTML các nút filter loại (`els.typeFilters`), gu (`els.purposeFilters`), select loại, hàng rating (range 0-5), nút ngày — sinh từ `TYPES`/`PURPOSES`/`DAY_OPTIONS`, escape bằng `escapeAttr`/`escapeHtml`.
- **`initMap()`**: `L.map("map",{zoomControl:false}).setView(DEFAULT_CENTER,13)`; thêm zoom control góc dưới phải; `L.tileLayer(OSM)`; `markerLayer = L.markerClusterGroup({...})` (fallback `L.layerGroup` nếu plugin chưa có); `map.on("click")` → nếu `pinMode` hoặc editor mở thì `setFormCoordinates` + `reverseLookup`.
- **`bindEvents()`**: phần dài nhất — gắn click/input/change cho mọi nút và dùng **event delegation** (gắn 1 listener cho container, dùng `event.target.closest("[data-...]")`). Ví dụ `els.typeFilters` click → toggle `activeTypes` → `syncFilterButtons()` + `render()`. Có listener `keydown` toàn cục: Tab → `trapEditorFocus`, Escape → đóng modal/lightbox/pinmode/editor theo thứ tự ưu tiên.

---

## PHẦN 3 — DỮ LIỆU QUÁN

**`loadPlaces()`**: đọc `STORAGE_KEY`; nếu trống → seed 5 quán (qua `normalizePlace`) và lưu; nếu có → `JSON.parse` → `.map(normalizePlace).filter(isValidPlace)`; lỗi → báo + trả `[]`.

**`savePlaces()`** (quan trọng — kiến trúc 2 tầng):
```js
const serialized = places.map(p => ({ ...p, images: p.images.map(i => ({id:i.id, category:i.category})) }));
localStorage.setItem(STORAGE_KEY, JSON.stringify(serialized));   // KHÔNG có dataUrl
persistImagesToDb();   // ảnh → IndexedDB
renderDataPanel();
```

**`normalizePlace(place)`**: trả object chuẩn — ép kiểu, `type` phải thuộc `TYPES` (nếu không → "cafe"), `priceLevel` clamp 1-4, `ratings` clamp 0-5 theo `PURPOSES`, `tags` trim + bỏ rỗng + tối đa 12, `images` qua `normalizeImages`, `openingHours` qua `normalizeOpeningHours`, `visits` qua `normalizeVisits`. **`isValidPlace`**: có `name` + `lat/lng` hữu hạn.

**`normalizeVisits`**: ép `{id,date,rating,note}`, lọc ngày đúng `YYYY-MM-DD`, sort giảm dần theo ngày, tối đa 50.

---

## PHẦN 4 — ẢNH & INDEXEDDB (đặc sắc)

```js
function imageDataUrl(image) { return image?.dataUrl || imageCache.get(image.id) || ""; }

function openImageDb() {            // mở DB, tạo store "images" nếu chưa có (onupgradeneeded)
  // trả promise cache lại trong imageDbPromise; nếu không có indexedDB → imageDbAvailable=false
}
async function loadImagesFromDb() { // đọc getAllKeys + getAll → imageCache.set(key,value); persistedImageIds.add(key)
}
function writeImageToDb(id, dataUrl) { /* put(dataUrl, id) trong transaction readwrite */ }
function deleteImageFromDb(id) { /* delete(id) + xóa khỏi cache/persistedImageIds */ }

function persistImagesToDb() {      // duyệt mọi ảnh có dataUrl, cache lại, ghi cái CHƯA persist
  places.forEach(p => p.images.forEach(img => {
    if (!img.dataUrl) return;
    imageCache.set(img.id, img.dataUrl);
    if (!persistedImageIds.has(img.id)) writes.push(writeImageToDb(img.id, img.dataUrl));
  }));
  return Promise.allSettled(writes);
}
function pruneOrphanImages() {      // ảnh trong DB/cache không còn quán tham chiếu → xóa
}
function hydratePlaceImages() {     // gắn dataUrl từ imageCache vào từng quán; bỏ ảnh không có dataUrl
}
async function initImages() { await loadImagesFromDb(); hydratePlaceImages(); await persistImagesToDb(); savePlaces(); render(); pruneOrphanImages(); }
```
➡️ **Câu chuyện 2 tầng**: localStorage giữ `{id,category}`; IndexedDB giữ `id→dataUrl`; `imageCache` là cầu nối RAM. Đây là phần đáng kể nhất khi phỏng vấn.

---

## PHẦN 5 — RENDER PIPELINE

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

- **`renderTagFilters`/`renderAreaFilters`**: đếm tần suất tag/khu vực, sort, dựng nút chip với `aria-pressed`.
- **`renderList(filtered)`**: dựng `.place-item` cho mỗi quán — thumbnail (từ `imageDataUrl`), điểm `getFitScore`, badge loại/giá/khoảng cách/trạng thái mở/ số lần ghé/ruột, dải mini-rating 5 tiêu chí, tag, nút Sửa/Ruột.
- **`renderMarkers(filtered)`**: **tái sử dụng marker** — xóa marker không còn trong `filtered`, thêm/cập nhật phần còn lại (`createMarkerIcon`, `createPopup`); chuyển markerLayer/heat tùy `heatmapMode`.
- **`renderHeat`**: nếu bật, dựng điểm `[lat,lng,weight=fitScore/100]` cho `L.heatLayer`.

---

## PHẦN 6 — LỌC, ĐIỂM SỐ, GỢI Ý

**`getFilteredPlaces()`**: `places.filter(...)` áp lần lượt: loại (`activeTypes`), giá (`priceRange`), khoảng cách (`maxDistanceKm` + `getDistanceFromUser`), khu vực, bộ sưu tập, đang mở cửa, gu (`activePurposes` mỗi cái rating ≥3), tag (mọi tag phải có), tìm tự do (chuẩn hóa text + `includes`). Rồi `.sort()` theo `els.sortMode.value` (fit/distance/recent/price/name).

**`getFitScore(place)`**:
```js
const keys = activePurposes.size ? [...activePurposes] : PURPOSES.map(p=>p.key);
const base = tổng rating theo keys / (keys.length*5);     // 0..1
return clamp(base*100 + (favorite?8:0) + recentBoost, 0, 100);
```

**`getMoodScore(place, mood)`**: switch theo mood — `work` = work*16 + power*10 + quiet*7 (+10 nếu tag wifi/ổ cắm); `date` = date*18 + taste*6 + quiet*5; `quiet` = quiet*20 + work*5; `taste` = taste*20 (+8 ruột); `nearby` = theo khoảng cách + 0.25*fit; `new` = chưa ghé thì 60 + 0.4*fit, ghé rồi thì giảm dần; `quick` = taste*12 + (food?18) + (giá≤2?10) + tag nhanh. Cuối: +6 nếu đang mở, +5 nếu ruột; clamp 0-100.

**`getRecommendedPlaces`/`pickWeightedRecommendations`**: lần đầu lấy top 3; reroll → **random có trọng số** `getRecommendationWeight = max(1,score)^1.25 * (vừa ghé ? 0.22 : 1)` → đa dạng mà ưu điểm cao, tránh quán mới ghé.

**`getFavoriteReminders`**: quán ruột mà `daysSince(lastVisit) >= STALE_FAVORITE_DAYS(30)` (hoặc chưa ghé) → top 3.

---

## PHẦN 7 — TAG MANAGER

**`renameTag(old, new)`**: `pushUndoSnapshot` → duyệt mọi quán đổi tag (chống trùng) → nếu có ảnh hưởng thì cập nhật `activeTags` + `savePlaces` + `render`; không thì `undoStack.pop()`. Trả số quán bị ảnh hưởng. **`deleteTag`** tương tự (xóa tag khỏi mọi quán). Cả hai có test kiểm chứng.

---

## PHẦN 8 — CRUD & EDITOR

**`savePlace(event)`**: `preventDefault` → gom giá trị form (tên/loại/giá/toạ độ/rating/tag/ghi chú/ngày/ruột/ảnh `editorPhotos`/giờ mở/lịch sử `editorVisits`) → `normalizePlace` → nếu không hợp lệ báo lỗi; `pushUndoSnapshot("sửa/thêm quán")`; cập nhật hoặc `unshift` vào `places`; `savePlaces()` + `closeEditor()` + `render()` + `selectPlace()` + `showUndoStatus`.

**`deleteCurrentPlace`**: `showConfirm` → `pushUndoSnapshot` → lọc bỏ khỏi `places` → save + render + undo toast.

**`selectPlace(id, panTo)`**: set `selectedId` + render; nếu `panTo` thì `markerLayer.zoomToShowLayer(marker, ()=>openPopup())` (tách cụm để thấy), không thì chỉ mở popup.

**`toggleFavorite`/`markPlaceVisited`**: đều `pushUndoSnapshot` + cập nhật + save + render + undo toast. `markPlaceVisited` chặn đánh dấu trùng trong cùng ngày.

**`parseCoordsString(raw)`**: bỏ `()`, đổi `;|`→`,`, regex bắt 2 số, validate lat∈[-90,90]/lng∈[-180,180]; sai → null. **`handleCoordsPaste`** dùng nó để tách vào 2 ô lat/lng.

**`resetFilters`**: đặt lại mọi biến filter về mặc định + đồng bộ UI + render.

---

## PHẦN 9 — KHÁM PHÁ (Overpass) & MẠNG

**`fetchWithRetry(url, opts, attempts=2)`**:
```js
for (i=0; i<attempts; i++) {
  res = await fetch(url, opts);
  if (res.status===429 || 503) { nếu còn lượt: await delay(900*(i+1)); continue; else: throw {rateLimited:true}; }
  return res;
} // lỗi network: delay(500) rồi thử lại
```

**`fetchNearbyPois(lat,lng,radius,limit)`**: dựng Overpass QL `[out:json][timeout:25];(node["amenity"~"${amenities}"]["name"](around:${radius},${lat},${lng}););out ${limit};`, POST qua `fetchWithRetry`, `parseOverpassPois`.

**`discoverAmenityFilter(category)`**: `cafe`→"cafe", `food`→"restaurant|fast_food", `bar`→"bar|pub", `dessert`→"ice_cream|bakery", mặc định = tất cả.

**`discoverArea`**: lấy `map.getCenter()` (hoặc userLocation), gọi `fetchNearbyPois` (tới 200), `showDiscoveredPlaces`. **`pickRandomNearby`**: tương tự nhưng ưu tiên POI chưa lưu; lỗi mạng → `fallbackRandomSaved()` (gợi ý từ danh sách đã lưu).

**`parseOverpassPois`**: lọc element có `lat/lon` hữu hạn + `tags.name`. `guessTypeFromTags`/`poiToDraft` (đã thấy qua test): đoán loại từ amenity, tạo nháp `{name,type,lat,lng}`.

---

## PHẦN 10 — ĐỊNH TUYẾN (OSRM)

**`fetchOsrmRoute(stops, key)`**: dựng URL `router.project-osrm.org/route/v1/driving/{lng,lat;...}?overview=full&geometries=geojson`, `fetchWithRetry`, `parseOsrmRoute`; gán `osrmRoute` rồi `renderRoute`/`renderItinerary`. Lỗi → bỏ qua (giữ đường chim bay).

**`parseOsrmRoute(data, key)`**: lấy `routes[0]`, trả `{key, distanceMeters, durationSec, latlngs}` (đảo `[lng,lat]`→`[lat,lng]` cho Leaflet).

**`formatDuration(sec)`**: <60 phút → "X phút", ≥60 → "Xh MM" hoặc "X giờ".

**`getItineraryDirectionsUrl(stops)`**: dựng URL Google Maps `dir`: origin = vị trí hiện tại (nếu có) hoặc điểm đầu, destination = điểm cuối, waypoints = giữa nối bằng `|`.

---

## PHẦN 11 — BACKUP MÃ HÓA (Web Crypto)

```js
async function encryptBackup(payload, password) {
  const salt = crypto.getRandomValues(new Uint8Array(16));
  const iv   = crypto.getRandomValues(new Uint8Array(12));
  const key  = await deriveBackupKey(password, salt);     // PBKDF2-SHA-256, 250000 vòng
  const cipher = await crypto.subtle.encrypt({name:"AES-GCM", iv}, key, encode(JSON));
  return { app, version:2, encrypted:true, algorithm:"AES-GCM", kdf:"PBKDF2-SHA-256",
           iterations:250000, salt(b64), iv(b64), data(b64) };
}
async function deriveBackupKey(password, salt, iterations=250000) {
  importKey(password, "PBKDF2") → deriveKey({name:"PBKDF2", salt, iterations, hash:"SHA-256"}, → AES-GCM 256)
}
```
`decryptBackup` đảo lại (đọc salt/iv/iterations từ file). Có test round-trip.

---

## PHẦN 12 — SERVICE WORKER (sw.js)

```
CACHE_VERSION = "taste-map-v36"
APP_SHELL = [file local + CDN Leaflet/markercluster/heat/lucide]
install  → cache.add từng asset (Promise.allSettled) → skipWaiting
activate → xóa cache ≠ CACHE_VERSION → clients.claim
message  → SKIP_WAITING → skipWaiting
fetch (GET):
  origin Nominatim/OSRM/Overpass → fetch thẳng
  tile.openstreetmap.org → cacheFirst(TILE_CACHE, 240)  // trimCache đệ quy xóa cũ
  navigate → networkFirstPage (rớt mạng → "./" hoặc index.html)
  app shell → staleWhileRevalidate(APP_SHELL_CACHE)
  còn lại → staleWhileRevalidate(RUNTIME_CACHE)
```

---

## PHẦN 13 — TESTS (tests/app.spec.js) — 47 ca

Gọi hàm app qua `page.evaluate`. Nhóm test: render & UI, import Google Maps, backup mã hóa round-trip, undo, PWA manifest, theme, đang-mở-cửa, thống kê, share payload, random nearby, parse Overpass POI, lọc giá/khu vực, lịch trình + OSRM parse, CSV, bộ sưu tập, heatmap, mood "new", nhắc ruột, lưu/đọc lịch trình, đổi/xóa tag, dialog accessibility (focus trap + Esc), ảnh album, i18n tĩnh+động, **ảnh IndexedDB + strip localStorage**, rehydrate sau reload, dọn ảnh mồ côi, lightbox, discover amenity filter, explore area (mock Overpass 25 marker), parse toạ độ, **lucide ghim version**, **fetchWithRetry 429**, modal không dùng prompt native, gist push/pull (mock).

---

## 🎯 Bảng tra cứu nhanh "hàm ↔ chức năng"

| Nhóm | Hàm chính |
|------|-----------|
| Khởi động | `init`, `cacheElements`, `hydrateStaticControls`, `initMap`, `bindEvents` |
| Dữ liệu | `loadPlaces`, `savePlaces`, `normalizePlace`, `isValidPlace`, `normalizeVisits/OpeningHours` |
| Ảnh/IndexedDB | `openImageDb`, `loadImagesFromDb`, `writeImageToDb`, `persistImagesToDb`, `hydratePlaceImages`, `pruneOrphanImages`, `imageDataUrl`, `initImages` |
| Render | `render`, `renderList/Markers/Heat/Route/Itinerary/Recommendations/Stats/TagFilters/AreaFilters/FavoriteReminder/DataPanel` |
| Lọc/điểm | `getFilteredPlaces`, `getFitScore`, `getMoodScore`, `getRecommendedPlaces`, `pickWeightedRecommendations`, `getFavoriteReminders`, `getOpeningStatus(At)` |
| Tag | `renameTag`, `deleteTag` |
| CRUD | `openEditor`, `savePlace`, `deleteCurrentPlace`, `selectPlace`, `toggleFavorite`, `markPlaceVisited`, `parseCoordsString`, `resetFilters` |
| Khám phá | `fetchNearbyPois`, `discoverArea`, `pickRandomNearby`, `parseOverpassPois`, `discoverAmenityFilter`, `guessTypeFromTags`, `poiToDraft` |
| Mạng | `fetchWithRetry`, `delayMs` |
| Định tuyến | `fetchOsrmRoute`, `parseOsrmRoute`, `formatDuration`, `getItineraryDirectionsUrl` |
| Backup | `encryptBackup`, `decryptBackup`, `deriveBackupKey`, `createBackupPayload`, `importCsvText`, `pushGistBackup`, `pullGistBackup` |
| i18n/theme | `t`, `applyTranslations`, `toggleLang`, `applyTheme`, `toggleTheme` |
| Tiện ích | `escapeHtml/escapeAttr`, `haversineMeters`, `showStatus/showUndoStatus/showConfirm/showPrompt` |

> 💡 Nắm bảng này + đọc file 08 là bạn có thể **chỉ tay vào bất kỳ hàm nào trong app.js và giải thích**.
