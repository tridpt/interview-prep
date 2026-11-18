# 4. Code chi tiết (giải thích từng file)

File này "mổ xẻ" từng file. Đọc kèm code thật trong `quan-quen-map/`. Phần đi sâu từng hàm app.js xem thêm file `08-code-walkthrough.md`.

---

## 4.1. `index.html` — Khung & DOM tĩnh (~574 dòng)

**Nhiệm vụ:** chứa toàn bộ DOM tĩnh — thanh công cụ, sidebar (filter, gợi ý, lịch trình, dữ liệu), khu bản đồ `#map`, panel danh sách + chi tiết, editor (form thêm/sửa quán dạng dialog), lightbox, modal.

**Điểm chính:**
- `<head>`: khai báo PWA (`manifest`, `theme-color`, apple metas) + nạp CSS Leaflet/markercluster.
- Cuối `<body>`: nạp thư viện CDN (Leaflet, markercluster, heat, Lucide) rồi **`i18n.js` → `app.js`** (thứ tự bắt buộc).
- Mọi text dịch gắn `data-i18n` / `data-i18n-placeholder` / `data-i18n-title` / `data-i18n-aria`.
- **Editor** (`#editorPanel`) là `role="dialog" aria-modal="true"` — quản lý focus (focus trap).

---

## 4.2. `i18n.js` — Từ điển dịch (~688 dòng)

- Định nghĩa `I18N_STRINGS = { vi: {...}, en: {...} }` — từ điển phẳng theo key (`"filters.title"`, `"ui.edit"`, `"msg.placeSaved"`...).
- Phần lớn file là **dữ liệu** (cặp key→chuỗi cho 2 ngôn ngữ). Logic dịch (`t`, `applyTranslations`) nằm trong `app.js`.
- Nạp **trước** app.js để `I18N_STRINGS` sẵn sàng.

---

## 4.3. `app.js` — Bộ não (~4.718 dòng)

Là **classic script** (không IIFE, không module) → mọi hàm ở global scope. Các nhóm:

### Hằng số & state (đầu file)
- Khóa lưu trữ (`STORAGE_KEY`, `IMAGE_DB_NAME`...), hằng cấu hình (`DEFAULT_TYPES`, `PURPOSES`, `DAY_OPTIONS`, `PHOTO_CATEGORIES`, `PRICE_ESTIMATE_VND`, `RANDOM_NEARBY_RADIUS`...), `seedPlaces` (5 quán mẫu).
- ~40 biến state toàn cục (xem file 03 mục 3.6).

### Khởi động & DOM
- `init()` (xem file 03), `cacheElements()` (gom ~150 id vào `els`), `hydrateStaticControls()` (dựng các nút filter/select/rating từ `TYPES`/`PURPOSES`/`DAY_OPTIONS`), `bindEvents()` (gắn toàn bộ sự kiện — phần dài nhất), `initMap()` (Leaflet + tile + markercluster + xử lý click bản đồ để ghim toạ độ).

### Lưu trữ & dữ liệu
- `loadPlaces()` (đọc localStorage, seed nếu trống), `savePlaces()` (strip ảnh khỏi localStorage + `persistImagesToDb()`), `normalizePlace()` / `isValidPlace()`, `normalizeVisits()`, `normalizeOpeningHours()`.

### Ảnh & IndexedDB
- `openImageDb()` (mở DB, tạo store `images`), `loadImagesFromDb()` (nạp vào `imageCache`), `writeImageToDb`/`deleteImageFromDb`, `persistImagesToDb()` (ghi ảnh mới chưa lưu), `hydratePlaceImages()` (gắn dataUrl từ cache vào quán), `pruneOrphanImages()` (xóa ảnh không còn tham chiếu), `imageDataUrl()`.

### Render (đồng bộ)
- `render()` điều phối; các render con: `renderList`, `renderMarkers`, `renderHeat`, `renderRoute`, `renderItinerary`, `renderCollections`, `renderPlaceDetail`, `renderStats`, `renderTagFilters`, `renderAreaFilters`, `renderRecommendations`, `renderFavoriteReminder`, `renderDataPanel`.

### Lọc, điểm số, gợi ý
- `getFilteredPlaces()` (áp mọi filter + sắp xếp 5 kiểu), `getFitScore()` (điểm hợp gu trên marker/list), `getMoodScore(place, mood)` (điểm theo mood gợi ý), `getRecommendedPlaces`/`pickWeightedRecommendations` (chọn 3 quán gợi ý, **reroll có trọng số** tránh quán mới ghé), `getFavoriteReminders()`.
- Trạng thái mở cửa: `getOpeningStatus()` / `getOpeningStatusAt(place, day, minutes)`. Khu vực: `getPlaceArea()`/`parseArea()`.

### CRUD & editor
- `openEditor`, `closeEditor`, `savePlace`, `deleteCurrentPlace`, `selectPlace`, `toggleFavorite`, `markPlaceVisited`, `setFormCoordinates`, `handleCoordsPaste`/`parseCoordsString`. Có **undo** qua `pushUndoSnapshot` + `showUndoStatus`.

### Bản đồ & khám phá
- `createMarkerIcon`, `createPopup`, `locateUser`, `toggleHeatmap`, `pickRandomNearby`, `discoverArea`, `fetchNearbyPois` (Overpass), `parseOverpassPois`, `guessTypeFromTags`, `poiToDraft`, `showDiscoveredPlaces`, `discoverAmenityFilter`.

### Định tuyến
- `fetchOsrmRoute`/`parseOsrmRoute` (OSRM), `renderRoute`, `getItineraryDirectionsUrl` (URL Google Maps đa điểm), `openExternalMap`/`getGoogleMapsUrl`/`getAppleMapsUrl`, `formatDuration`.

### Dữ liệu (backup/import)
- `createBackupPayload`, `exportPlainBackup`, `exportEncryptedBackup`, `encryptBackup`/`decryptBackup`/`deriveBackupKey` (Web Crypto), `importPlaces`, `importCsvText`/`parseCsv`/`exportCsv`, `saveGistSettings`/`pushGistBackup`/`pullGistBackup`, `importFromGoogleMaps`/parse URL.

### i18n, theme, tiện ích
- `t`, `applyTranslations`, `toggleLang`, `applyTheme`, `toggleTheme`, `typeLabel`, `purposeLabel`; `escapeHtml`/`escapeAttr`, `haversineMeters`, `fetchWithRetry`, `showStatus`/`showUndoStatus`/`showConfirm`/`showPrompt` (modal/toast trong app, **không dùng `prompt()` native**).

---

## 4.4. `sw.js` — Service Worker (~148 dòng)

Đã mô tả 4 chiến lược ở file 03 mục 3.7. Các hàm: `networkFirstPage`, `staleWhileRevalidate`, `cacheFirst`, `trimCache` (giới hạn số tile), `isAppShellRequest`, `isCacheable`. `CACHE_VERSION = "taste-map-v36"`.

> 💡 **Câu hỏi hay gặp:** "Vì sao Nominatim/Overpass/OSRM fetch thẳng không cache?" → dữ liệu động/theo thời gian + điều khoản dịch vụ; cache sẽ trả kết quả cũ và lãng phí. Chỉ cache **tile** (ít đổi) để xem lại offline.

---

## 4.5. `tests/app.spec.js` — 47 test Playwright (~774 dòng)

Đây là **điểm nhấn của dự án**. Vài ví dụ test thật:
- `app render` — kiểm tra render 5 quán seed, 3 gợi ý, lọc theo tag, thu sidebar.
- `encrypted backup can decrypt to original payload` — mã hóa rồi giải mã, so khớp.
- `undo restores data after adding a place` — thêm quán → bấm Hoàn tác → còn 5 quán.
- `images are stored in indexeddb and stripped from localStorage` — kiểm tra **đúng** kiến trúc 2 tầng.
- `fetchWithRetry retries on 429 then surfaces a rate-limit error` — mock fetch trả 429, kiểm thử lại + cờ `rateLimited`.
- `gist push/pull uses mocked fetch` — mock GitHub API.
- `lucide is pinned to a fixed version (not latest)` — chống rủi ro thư viện đổi bất ngờ.
- Các test i18n, theme, lọc giá/khu vực/đang mở, lịch trình + OSRM, CSV, bộ sưu tập, heatmap, lightbox, dialog accessibility, khám phá Overpass...

**Cách test gọi hàm:** `page.evaluate(() => someAppFunction())` chạy code **trong trang** → gọi trực tiếp hàm global của app. Test mạng thì `window.fetch = async () => ({...})` để mock.

---

## 4.6. `.github/workflows/ci.yml` — CI/CD

Push lên `main` → job `checks` (`npm run check` + `npm test`) → nếu pass, job `deploy-pages` deploy GitHub Pages. **Không pass thì không deploy** → tránh đẩy bản hỏng.

---

## 4.7. Tổng kết các quyết định kỹ thuật

| Quyết định | Lý do |
|-----------|-------|
| Vanilla JS, không bundler | Deploy = file tĩnh; test gọi thẳng hàm global; nhẹ |
| localStorage + IndexedDB | Text nhanh/đơn giản; ảnh lớn không làm đầy localStorage 5MB |
| `render()` trung tâm (state→UI) | Nhất quán, dễ suy luận, không cần framework |
| 4 chiến lược cache theo origin | Dữ liệu động fetch thẳng; tile cache-first; trang network-first |
| `fetchWithRetry` + fallback | Bền bỉ với dịch vụ dùng chung (429/503); OSRM lỗi → đường chim bay |
| Backup AES-GCM + PBKDF2 250k | Bảo mật dữ liệu khi chia sẻ/đồng bộ |
| Modal/toast tự viết (không `prompt()`) | UX nhất quán + test được (có test chặn `prompt` native) |
| 47 test Playwright + CI | Giữ chất lượng, chặn deploy bản hỏng |
| escapeHtml mọi dữ liệu (kể cả OSM) | Chống XSS |
| Ghim phiên bản CDN | Tránh thư viện đổi bất ngờ phá app |
