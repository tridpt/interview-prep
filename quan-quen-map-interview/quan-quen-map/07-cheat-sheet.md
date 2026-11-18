# 7. Cheat Sheet — Xem nhanh trước giờ G

Đọc lướt 5-10 phút trước khi vào phỏng vấn.

---

## 🎯 Pitch 30 giây (học thuộc)
> Quán quen: web app **PWA** lưu quán ăn/cafe **theo gu cá nhân** trên bản đồ — chấm điểm 5 tiêu chí riêng, gợi ý "hôm nay đi đâu" theo mood, lập lịch trình tuyến đường thật, khám phá quán mới từ OpenStreetMap. **Vanilla JS + Leaflet, không backend, không bundler**. Dữ liệu ở **localStorage (text) + IndexedDB (ảnh)**, backup **mã hóa AES-GCM**, đồng bộ **GitHub Gist** tùy chọn. Có **47 test Playwright + CI/CD** (chỉ deploy khi pass), song ngữ VI/EN, chạy offline.

## 🧱 Tech stack
- **Core**: Vanilla HTML/CSS/JS (ES2020), không bundler.
- **Bản đồ**: Leaflet 1.9.4 + markercluster + leaflet.heat; Lucide icons (CDN ghim version).
- **Lưu trữ**: localStorage (text) + IndexedDB (ảnh) + Web Crypto (backup).
- **Dịch vụ**: OSM tiles, Nominatim, Overpass, OSRM, GitHub Gist.
- **Test/CI**: Playwright (47 test) + http-server + GitHub Actions.

## 🗂️ File
| File | Vai trò |
|---|---|
| index.html | DOM tĩnh; nạp i18n.js → app.js + CDN |
| i18n.js | I18N_STRINGS vi/en |
| app.js (~4718) | Bộ não: state, init, render, lọc/điểm, CRUD, bản đồ, khám phá, định tuyến, backup |
| style.css (~2279) | theme + dark mode + responsive |
| sw.js | service worker, 4 chiến lược cache |
| tests/app.spec.js | 47 test Playwright |

## 📐 Kiến trúc
Client-only, **không backend**. **render()** trung tâm: state → UI. localStorage (quán text) + IndexedDB (ảnh). Gọi dịch vụ bản đồ qua fetch (có fetchWithRetry).

## 💾 Lưu trữ 2 tầng (NHỚ KỸ)
Ảnh nặng → **IndexedDB**; text → **localStorage** (images chỉ {id,category}, KHÔNG dataUrl). Khởi động: loadImagesFromDb → imageCache → hydratePlaceImages. pruneOrphanImages dọn ảnh mồ côi. Lý do: localStorage ~5MB.

## 🗺️ Leaflet
tileLayer (OSM, z/x/y) · marker + popup · markercluster (gom cụm) · leaflet.heat (heatmap). map.invalidateSize() khi đổi kích thước container.

## 🌐 Dịch vụ ngoài
OSM tiles · Nominatim (địa chỉ↔toạ độ) · Overpass (tìm quán: `node["amenity"~...]["name"](around:r,lat,lng)`) · OSRM (định tuyến) · Gist (đồng bộ).
**fetchWithRetry**: 429/503 → đợi rồi thử lại → cờ rateLimited. Fallback: OSRM lỗi→chim bay; khám phá lỗi→danh sách đã lưu.

## 🔐 Backup mã hóa
PBKDF2-SHA-256 **250.000 vòng** + salt → khóa; **AES-GCM** + iv ngẫu nhiên. File: salt/iv/data base64. Web Crypto API.

## 🧠 render() pipeline
getFilteredPlaces() → renderList/Markers/Heat/Route/Itinerary/Recommendations/Stats. Marker **tái sử dụng** (so Map id, chỉ thêm/xóa/cập nhật khác biệt).

## 🎯 Gợi ý theo mood
getMoodScore(place, mood): work/date/quiet/taste/nearby/new/quick. Reroll **random có trọng số** (score^1.25, quán vừa ghé ×0.22).

## 🧪 Test & CI (ĐIỂM NHẤN)
**47 test Playwright** E2E; gọi hàm global qua page.evaluate; test mạng **mock window.fetch**. check-encoding.js chống mojibake. **GitHub Actions**: check + test → chỉ deploy Pages khi pass.

## 📦 PWA
manifest + sw.js. Cache: app shell + CDN + tile (cache-first, trim 240). 4 chiến lược theo origin. **share_target** nhận link từ điện thoại. Nút Cài app / Cập nhật (SKIP_WAITING).

## 🔐 Bảo mật & riêng tư
Dữ liệu ở máy, không gửi đi (trừ Gist). escapeHtml mọi dữ liệu (kể cả OSM). Modal/toast tự viết, không prompt() native. Ghim version CDN.

## ⚖️ Trade-off
No-backend → riêng tư/đơn giản, đổi lại đồng bộ thủ công (Gist). No-bundler → deploy dễ, test gọi hàm global, đổi lại app.js lớn.

## 🚀 Cải thiện tương lai
Tách ES modules + bundler; test a11y screen reader; cache Overpass theo ô lưới; backend tùy chọn cho sync realtime.

## 💬 Thuật ngữ chốt
- **PWA / Service Worker / Cache API / share_target**
- **localStorage vs IndexedDB / imageCache / hydrate / orphan prune**
- **Leaflet / tile / marker / markercluster / heatmap**
- **Nominatim / Overpass / OSRM / Gist / POI / amenity**
- **fetchWithRetry / 429 / fallback / resilience**
- **Web Crypto / PBKDF2 / AES-GCM / salt / iv**
- **render() = f(state) / undo stack / normalizePlace**
- **Playwright E2E / mock fetch / GitHub Actions CI/CD**

---

### ✅ Checklist trước phỏng vấn
- [ ] Pitch dự án 30 giây.
- [ ] Giải thích localStorage + IndexedDB & luồng ảnh.
- [ ] Mô tả 47 test + CI/CD chặn deploy bản hỏng.
- [ ] Backup mã hóa (PBKDF2 + AES-GCM).
- [ ] fetchWithRetry + fallback mạng.
- [ ] render() = f(state).
- [ ] 2 hạn chế + cách khắc phục.
- [ ] 2 câu hỏi ngược nhà tuyển dụng.

**Hít thở sâu. Bạn hiểu dự án này. Tự tin kể câu chuyện của mình. 🍀**
