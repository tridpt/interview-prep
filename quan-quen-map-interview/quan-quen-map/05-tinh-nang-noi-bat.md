# 5. Phân tích sâu các tính năng "ăn điểm"

File này giúp bạn **kể chuyện ấn tượng**. Mỗi tính năng kèm: ý tưởng, cách làm, "vì sao hay".

---

## 5.1. Lưu trữ hai tầng: localStorage + IndexedDB ⭐ (rất hay được hỏi)

**Ý tưởng:** dữ liệu text nhẹ → localStorage; ảnh nặng → IndexedDB.

**Cách làm:** `savePlaces()` strip `dataUrl` khỏi quán trước khi ghi localStorage (chỉ giữ `{id, category}`); ảnh ghi vào IndexedDB qua `persistImagesToDb()`. Khi khởi động, `loadImagesFromDb()` → `imageCache` (Map RAM) → `hydratePlaceImages()` gắn lại dataUrl. `pruneOrphanImages()` dọn ảnh không còn quán tham chiếu.

**Vì sao hay:** thể hiện hiểu **giới hạn localStorage (~5MB)** và biết chọn đúng kho cho đúng loại dữ liệu. Có hẳn test kiểm chứng (`images are stored in indexeddb and stripped from localStorage`). Đây là câu hỏi kinh điển — trả lời tốt là điểm cộng lớn.

---

## 5.2. Tích hợp dịch vụ bản đồ thật + phòng thủ mạng ⭐

**Ý tưởng:** không tự host địa điểm — dùng OpenStreetMap (tile + Overpass), Nominatim (địa chỉ), OSRM (định tuyến).

**Cách làm & điểm phòng thủ:**
- `fetchWithRetry`: thử lại khi 429/503, ném lỗi có cờ `rateLimited` để UI báo rõ.
- **Fallback nhiều tầng**: OSRM lỗi → giữ **đường chim bay**; "Random gần tôi" lỗi mạng → gợi ý từ **danh sách đã lưu**; offline → vẫn dùng quán đã lưu.
- Overpass query dựng động theo loại quán (`discoverAmenityFilter` map "food" → `restaurant|fast_food`).

**Vì sao hay:** cho thấy tư duy **resilience** — phụ thuộc dịch vụ miễn phí dùng chung nhưng không để app sập khi chúng chậm/giới hạn.

---

## 5.3. Backup mã hóa bằng Web Crypto ⭐

**Ý tưởng:** xuất file backup an toàn, cần mật khẩu mới mở.

**Cách làm:** `encryptBackup` dùng **PBKDF2-SHA-256 250.000 vòng** (dẫn khóa từ mật khẩu + salt ngẫu nhiên) → **AES-GCM** với iv ngẫu nhiên; file lưu salt/iv/data base64 + metadata. `decryptBackup` đảo lại.

**Vì sao hay:** kiến thức bảo mật thực chiến (KDF chống brute-force, AES-GCM vừa mã hóa vừa xác thực toàn vẹn, salt/iv ngẫu nhiên mỗi lần). Có test round-trip mã hóa↔giải mã.

---

## 5.4. Render pipeline "state → UI" không framework ⭐

**Ý tưởng:** mọi thay đổi state → gọi **một** `render()` vẽ lại toàn bộ UI từ state.

**Cách làm:** `render()` tính `getFilteredPlaces()` rồi gọi ~12 render con (list, markers, heat, route, recommendations, stats...). Marker được **tái sử dụng** (so `markers` Map với id đang hiển thị, chỉ thêm/xóa/cập nhật phần khác biệt) thay vì vẽ lại từ đầu.

**Vì sao hay:** tái hiện tinh thần "UI là hàm của state" (như React) bằng vanilla — dễ suy luận, ít bug trạng thái. Việc tái sử dụng marker cho thấy quan tâm hiệu năng.

---

## 5.5. Gợi ý "Hôm nay đi đâu" theo mood + reroll có trọng số ⭐

**Ý tưởng:** chọn 3 quán hợp tâm trạng hiện tại, bấm reroll ra gợi ý mới nhưng "thông minh".

**Cách làm:** `getMoodScore(place, mood)` cho điểm theo từng mood (work ưu tiên ổ cắm/yên tĩnh + tag wifi; date ưu tiên điểm date; nearby theo khoảng cách; new ưu tiên quán **chưa ghé**...). Reroll dùng **random có trọng số** (`pickWeightedRecommendations`): điểm cao → trọng số cao (`score^1.25`), quán **vừa ghé gần đây bị phạt** trọng số (×0.22) → đa dạng mà vẫn hợp lý.

**Vì sao hay:** thuật toán gợi ý có chủ đích (không random thuần), cân bằng "hợp gu" và "đa dạng/mới mẻ".

---

## 5.6. 47 test Playwright + CI/CD ⭐⭐ (điểm khác biệt lớn nhất)

**Ý tưởng:** đảm bảo chất lượng tự động, không deploy bản hỏng.

**Cách làm:** Playwright điều khiển trình duyệt thật kiểm tra mọi luồng (render, import, backup, undo, theme, i18n, lọc, lịch trình, OSRM, CSV, IndexedDB, Gist mock...). Test mạng **mock `window.fetch`**. CI GitHub Actions chạy check + test, **chỉ deploy Pages khi pass**. Còn có `check-encoding.js` chống mojibake và test "lucide phải ghim version".

**Vì sao hay:** rất hiếm ở dự án cá nhân. Thể hiện **tư duy kỹ sư trưởng thành**: viết test cho cả happy path lẫn lỗi (429), mock dịch vụ ngoài, chặn regression. Nếu được hỏi "làm sao đảm bảo không hỏng khi thêm tính năng?" → đây là câu trả lời.

---

## 5.7. PWA offline-first cho bản đồ + share_target

**Ý tưởng:** cài như app, xem lại quán đã lưu kể cả offline; nhận link chia sẻ từ điện thoại.

**Cách làm:** SW precache app shell + thư viện CDN; **cache tile đã xem** (cache-first, giới hạn 240); `share_target` trong manifest nhận link Google Maps chia sẻ sang app để import.

**Vì sao hay:** caching tile bản đồ là bài toán riêng (số lượng lớn → cần `trimCache`); `share_target` là tính năng PWA nâng cao ít người biết.

---

## 5.8. Khám phá quán thật từ OpenStreetMap

**Ý tưởng:** không chỉ quản lý quán đã lưu mà còn **tìm quán mới** quanh bạn.

**Cách làm:** `discoverArea`/`pickRandomNearby` gọi Overpass quanh tâm bản đồ (bán kính 1-10km), `parseOverpassPois` lọc POI có toạ độ + tên, `guessTypeFromTags` đoán loại từ tag OSM, `poiToDraft` biến thành nháp điền sẵn form để lưu nhanh. Marker khám phá riêng (✨).

**Vì sao hay:** biến app từ "sổ tay tĩnh" thành công cụ khám phá, dùng truy vấn Overpass QL thật.

---

## 5.9. Bảng tổng hợp "tính năng ↔ khái niệm kỹ thuật" (học thuộc)

| Tính năng | Khái niệm bạn thể hiện |
|-----------|------------------------|
| Lưu trữ 2 tầng | localStorage vs IndexedDB, giới hạn 5MB, async storage |
| Dịch vụ bản đồ + retry | REST, fetchWithRetry, rate limit 429/503, fallback, resilience |
| Backup mã hóa | Web Crypto, PBKDF2, AES-GCM, salt/iv |
| render() trung tâm | "UI = f(state)", tái sử dụng marker, hiệu năng |
| Gợi ý theo mood | thuật toán chấm điểm + random có trọng số |
| 47 test + CI/CD | Playwright E2E, mock fetch, GitHub Actions, chặn regression |
| PWA offline | service worker, cache tile, share_target |
| Khám phá OSM | Overpass QL, geocoding, normalize dữ liệu ngoài |
| Leaflet | tile/marker/cluster/heat, geospatial |

> 💡 **Chiến lược kể chuyện:** nếu được hỏi "tính năng/điểm em tự hào nhất?", chọn **47 test + CI/CD** (độ chín kỹ thuật) hoặc **lưu trữ 2 tầng** (giải quyết giới hạn thật). Cả hai đều cho thấy bạn nghĩ xa hơn "chạy được".
