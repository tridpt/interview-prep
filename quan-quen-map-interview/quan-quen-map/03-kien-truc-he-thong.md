# 3. Kiến trúc hệ thống

## 3.1. Sơ đồ tổng thể

```
┌───────────────────────────────── TRÌNH DUYỆT ─────────────────────────────────┐
│                                                                                │
│  index.html ──▶ DOM tĩnh + data-i18n                                           │
│  i18n.js   ──▶ I18N_STRINGS (vi/en) ──▶ t()                                    │
│  app.js    ──▶ init() ─▶ state toàn cục ─▶ render() ─▶ DOM + Leaflet            │
│  style.css ──▶ theme (light/dark) + responsive                                 │
│        │              │                  │                                     │
│        ▼              ▼                  ▼                                     │
│  localStorage    IndexedDB          Leaflet map                                │
│  (quán text,     (ảnh quán)         (tiles, markers, cluster, heat)            │
│   cấu hình)                                                                    │
│        │                                  │  (fetch, có fetchWithRetry)        │
│        ▼                                  ▼                                     │
│   sw.js (PWA cache)            Nominatim · Overpass · OSRM · GitHub Gist        │
└────────────────────────────────────────────────────────────────────────────────┘
```

**Đặc điểm cốt lõi:** app **hoàn toàn phía client, không backend**. Mọi logic + state ở trình duyệt. Khác cap-cuu-101 ở chỗ: dữ liệu **động** (gọi nhiều dịch vụ bản đồ ngoài), lưu trữ **hai tầng** (localStorage + IndexedDB), và có **kiểm thử tự động + CI/CD**.

## 3.2. Phân lớp & trách nhiệm các file

| File | Vai trò |
|------|---------|
| `index.html` | Khung & toàn bộ DOM tĩnh; nạp `i18n.js` → `app.js` + thư viện CDN |
| `i18n.js` | Từ điển dịch `I18N_STRINGS` (vi/en) — nạp **trước** app.js |
| `app.js` | Bộ não: state, init, render, lọc/điểm, CRUD, bản đồ, khám phá, định tuyến, backup, i18n, theme |
| `style.css` | Theme (biến CSS + dark mode), layout grid, responsive |
| `sw.js` | Service worker: cache app shell + tile + runtime với 4 chiến lược theo origin |
| `manifest.json` | PWA: tên, icon, `share_target` |
| `tests/app.spec.js` | 47 test Playwright |
| `.github/workflows/ci.yml` | CI: check + test → deploy Pages |

> 💡 **Không bundler:** `index.html` nạp trực tiếp 3 file JS theo thứ tự `i18n.js → app.js` + CDN. Mọi hàm trong `app.js` ở **global scope** → test Playwright gọi được trực tiếp qua `page.evaluate`.

## 3.3. Luồng khởi động (`init()`) — học kỹ

`app.js` đăng ký `DOMContentLoaded → init()`. Thứ tự:

```
1. cacheElements()        — gom mọi phần tử DOM theo id vào object `els`
2. applyStoredLang()      — đọc ngôn ngữ đã lưu, dịch UI tĩnh
3. applyStoredTheme()     — áp dark/light
4. loadTypes()            — nạp loại quán (mặc định + tùy chỉnh)
5. setSidebarCollapsed()  — khôi phục trạng thái thanh bên
6. hydrateStaticControls()— dựng control động (filter loại/gu, select, rating, ngày)
7. syncRangeLabels(); places=loadPlaces(); collections=...; savedItineraries=...
8. initMap()              — khởi tạo Leaflet, tile, cluster/heat layer
9. bindEvents()           — gắn TOÀN BỘ sự kiện
10. render() + renderDataPanel() + refreshIcons()
11. setupInstallPrompt(); registerServiceWorker(); handleSharedLaunch()
12. initImages()          — ASYNC: nạp ảnh từ IndexedDB → hydrate → persist → dọn mồ côi
13. maybeShowDiscoverHint()— gợi ý lần đầu mở app
```

**Điểm đáng nói:** `loadPlaces()` nếu chưa có dữ liệu sẽ **seed 5 quán mẫu** (ở TP.HCM). `initImages()` chạy **bất đồng bộ** sau cùng vì IndexedDB là async.

## 3.4. Render pipeline — hàm `render()` trung tâm

Toàn bộ UI được vẽ lại từ state qua **một hàm `render()`** (mô hình "state → render", giống tinh thần React nhưng thủ công):

```
render():
  filtered = getFilteredPlaces()        // áp mọi bộ lọc + sắp xếp
  renderTagFilters / renderTagManager
  renderAreaFilters
  renderRecommendations                 // "hôm nay đi đâu" theo mood
  renderFavoriteReminder                // nhắc quán ruột lâu chưa ghé
  renderList(filtered)                  // danh sách quán bên trái
  renderMarkers(filtered)               // marker trên bản đồ
  renderHeat(filtered)                  // heatmap (nếu bật)
  renderRoute / renderItinerary         // tuyến đường + lịch trình
  renderCollections / renderPlaceDetail / renderStats
  refreshIcons()                        // Lucide vẽ lại icon
```

➡️ **Triết lý:** mọi thay đổi state (lọc, chọn quán, thêm quán...) → gọi `render()` → UI nhất quán với state. Đơn giản, dễ suy luận, không cần framework.

## 3.5. Mô hình dữ liệu

### Place (quán)
```js
{
  id, name, type,              // type: cafe|food|sweet|drink|c-<custom>
  address, lat, lng,
  priceLevel: 1..4,
  ratings: { taste, quiet, power, date, work },  // mỗi mục 0..5
  tags: string[],              // tối đa 12
  notes, lastVisit, favorite,
  images: [{ id, category }],  // category: food|space|menu|other — KHÔNG chứa dataUrl trong localStorage
  openingHours: { days:[0..6], open:"HH:MM", close:"HH:MM" },
  visits: [{ id, date, rating, note }],
  createdAt, updatedAt
}
```
- `normalizePlace()` chuẩn hóa mọi quán (điền mặc định, lọc hợp lệ); `isValidPlace()` yêu cầu có tên + lat/lng hữu hạn.
- Ảnh: trong RAM có thêm `dataUrl`; trong localStorage chỉ `{id, category}`; trong IndexedDB `id → dataUrl`.

### Khóa lưu trữ (localStorage)
| Khóa | Nội dung |
|------|----------|
| `quan-quen-map:places:v1` | Danh sách quán (ảnh đã strip) |
| `:collections:v1` `:itineraries:v1` `:types:v1` | Bộ sưu tập / lịch trình / loại tùy chỉnh |
| `:theme:v1` `:lang:v1` `:sidebar-state:v1` | Tùy chọn giao diện |
| `:gist-settings:v1` `:backup-meta:v1` | Đồng bộ & backup |
| `:discover-hint:v1` | Đã hiện gợi ý lần đầu chưa |

**IndexedDB:** DB `quan-quen-map-images`, store `images` (key = id ảnh, value = dataUrl).

## 3.6. State toàn cục (trong app.js)

Các biến chính: `places`, `selectedId`, `activeTypes/activePurposes/activeTags/activeAreas` (đều là `Set`), `openNowOnly`, `priceRange`, `maxDistanceKm`, `itinerary`, `collections`, `savedItineraries`, `userLocation`, `map`, `markers` (Map), `markerLayer`, `heatLayer`, `discoverMarkers`, `imageCache` (Map), `lang`, `undoStack`, ...

## 3.7. Tầng offline — Service Worker (4 chiến lược)

```
install  → precache APP_SHELL (file local + thư viện CDN Leaflet/lucide) → skipWaiting
activate → xóa cache khác CACHE_VERSION → clients.claim
message  → nhận SKIP_WAITING (khi user bấm "Cập nhật")
fetch    → phân loại theo origin:
   Nominatim/Overpass/OSRM  → fetch thẳng (không cache)
   tile.openstreetmap.org   → cache-first (giới hạn 240 tile, trimCache)
   navigate (trang)         → network-first
   app shell                → stale-while-revalidate
   còn lại (runtime)        → stale-while-revalidate
```
Đổi asset phải **bump `CACHE_VERSION`** (vd v36 → v37). `Promise.allSettled` khi precache để 1 file lỗi không hỏng cả cài đặt.

## 3.8. Vì sao kiến trúc này hợp lý?

- **Riêng tư & offline-first**: dữ liệu ở máy, chạy offline với quán đã lưu.
- **Hai tầng lưu trữ**: text ở localStorage (nhanh, đơn giản), ảnh ở IndexedDB (dung lượng lớn) → không đụng giới hạn 5MB.
- **Phụ thuộc dịch vụ dùng chung có phòng thủ**: `fetchWithRetry` + fallback (OSRM lỗi → đường chim bay; khám phá lỗi → gợi ý từ danh sách đã lưu).
- **Chất lượng được bảo chứng**: 47 test + CI chặn deploy bản hỏng — hiếm thấy ở dự án cá nhân, là điểm nhấn lớn khi phỏng vấn.
