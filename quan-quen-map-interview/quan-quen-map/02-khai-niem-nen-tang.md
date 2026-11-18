# 2. Các khái niệm nền tảng (lý thuyết)

File này giải thích **mọi thuật ngữ** trong dự án, cho người chưa biết gì. Nắm chắc phần này sẽ giúp bạn tự tin trước mọi câu hỏi "khái niệm".

---

## 2.1. PWA & Service Worker (tóm tắt)

**PWA (Progressive Web App)**: web app cài được lên màn hình chính và chạy offline như app native. Cần 3 thứ: HTTPS (hoặc localhost), **Web App Manifest** (`manifest.json`), và **Service Worker**.

**Service Worker (SW)**: script chạy nền, đóng vai **proxy mạng phía client** — chặn request và quyết định trả từ cache hay đi mạng. Vòng đời: `install` (precache) → `activate` (dọn cache cũ) → `fetch` (chặn request).

**Chi tiết riêng của dự án này** (xem kỹ ở file 04): SW phân loại request theo **origin**:
- Nominatim / Overpass / OSRM → **fetch thẳng** (dữ liệu động, không cache).
- Tile OpenStreetMap → **cache-first** có giới hạn 240 tile (`trimCache`).
- Điều hướng trang → **network-first**.
- App shell / runtime → **stale-while-revalidate**.

**`share_target`** trong manifest: cho phép khi cài như PWA, app **nhận link được chia sẻ** từ điện thoại (ví dụ chia sẻ một địa điểm Google Maps sang Quán quen).

---

## 2.2. Leaflet — thư viện bản đồ

**Leaflet là gì?** Thư viện JavaScript mã nguồn mở để hiển thị **bản đồ tương tác**. Nó không tự có dữ liệu bản đồ — nó hiển thị các **tile** (ô ảnh bản đồ) lấy từ nhà cung cấp (ở đây là OpenStreetMap).

**Các khái niệm Leaflet trong dự án:**
- **Tile layer** (`L.tileLayer`): lớp ảnh nền bản đồ, URL dạng `tile.openstreetmap.org/{z}/{x}/{y}.png` (`z`=zoom, `x/y`=toạ độ ô).
- **Marker** (`L.marker`): chấm đánh dấu một quán; có icon tùy biến (`createMarkerIcon`) và popup (`createPopup`).
- **Layer group / markercluster**: gom nhiều marker. Khi nhiều quán gần nhau, **markercluster** gộp thành một cụm số, bấm vào tách ra → tránh rối bản đồ.
- **Heat layer** (`leaflet.heat`): vẽ **bản đồ nhiệt (heatmap)** thể hiện mật độ quán bằng vùng màu thay cho marker.
- **`map.setView([lat,lng], zoom)`**, **`map.getCenter()`**, **`map.invalidateSize()`** (vẽ lại khi đổi kích thước container, ví dụ khi thu/mở sidebar).

**Tile là gì (nói thêm):** bản đồ được chia thành lưới các ô ảnh 256×256px theo từng mức zoom; trình duyệt chỉ tải các ô đang nhìn thấy → nhẹ và nhanh. Dự án **cache các tile đã xem** để xem lại offline.

---

## 2.3. localStorage vs IndexedDB (rất quan trọng)

Cả hai đều lưu dữ liệu **trong trình duyệt**, còn sau khi đóng tab. Nhưng khác nhau:

| | localStorage | IndexedDB |
|---|---|---|
| Kiểu dữ liệu | Chỉ **chuỗi text** (phải `JSON.stringify`) | Object, blob, dữ liệu lớn |
| Dung lượng | ~5MB | Lớn hơn nhiều (hàng trăm MB) |
| API | Đồng bộ, đơn giản | **Bất đồng bộ** (event/promise), phức tạp hơn |

**Vì sao dự án dùng CẢ HAI?** Đây là một **quyết định kiến trúc đáng nói**:
- **Ảnh quán** (base64, rất nặng) → lưu trong **IndexedDB** (kho `images`, key = id ảnh, value = dataUrl).
- **Dữ liệu quán dạng text** (tên, toạ độ, điểm, tag...) → lưu trong **localStorage**, nhưng trường `images` **chỉ giữ `{id, category}`, KHÔNG giữ dataUrl** → tránh localStorage bị đầy 5MB.
- Khi khởi động: `loadImagesFromDb()` nạp ảnh từ IndexedDB vào một `imageCache` (Map trong RAM), rồi `hydratePlaceImages()` gắn dataUrl trở lại vào từng quán.
- **Dọn ảnh mồ côi** (`pruneOrphanImages`): xóa ảnh trong IndexedDB không còn quán nào tham chiếu (ví dụ sau khi xóa quán).

> 💡 Đây là câu hỏi phỏng vấn rất hay: "Vì sao tách ảnh sang IndexedDB?" → vì localStorage giới hạn ~5MB và lưu ảnh base64 sẽ đầy ngay; IndexedDB chứa được nhiều và không chặn luồng chính (async).

---

## 2.4. Web Crypto API & backup mã hóa

**Web Crypto API** (`crypto.subtle`): API mã hóa chuẩn của trình duyệt. Dự án dùng nó cho **backup mã hóa**:
- **PBKDF2** (Password-Based Key Derivation Function 2): biến **mật khẩu người dùng** thành một khóa mã hóa, lặp **250.000 vòng** với SHA-256 + một `salt` ngẫu nhiên → chống dò mật khẩu (brute force) chậm hơn rất nhiều.
- **AES-GCM**: thuật toán mã hóa đối xứng (vừa mã hóa vừa xác thực toàn vẹn), dùng một `iv` (initialization vector) ngẫu nhiên.
- File backup mã hóa lưu: `salt`, `iv`, `data` (đã mã hóa) dạng base64, kèm metadata (`iterations`, `algorithm`). Khôi phục cần đúng mật khẩu.

**Thuật ngữ:**
- **Salt**: chuỗi ngẫu nhiên thêm vào trước khi băm/dẫn khóa → cùng mật khẩu nhưng salt khác thì khóa khác → chống bảng tra cứu (rainbow table).
- **IV (Initialization Vector)**: giá trị ngẫu nhiên để cùng dữ liệu + cùng khóa vẫn ra bản mã khác nhau.

> 💡 Nói được "PBKDF2 250k vòng + AES-GCM, salt và iv ngẫu nhiên mỗi lần" là một điểm cộng lớn về kiến thức bảo mật.

---

## 2.5. Các dịch vụ bản đồ ngoài

Dự án không tự host dữ liệu địa điểm — nó gọi các **API miễn phí dùng chung**:
- **OpenStreetMap (OSM)**: dự án bản đồ mở (như "Wikipedia của bản đồ"). Cung cấp tile nền.
- **Nominatim**: dịch vụ **geocoding** của OSM — đổi *tên/địa chỉ → toạ độ* (search) và *toạ độ → địa chỉ* (reverse). Dự án dùng `reverseLookup` để điền địa chỉ khi bấm bản đồ.
- **Overpass API**: truy vấn dữ liệu OSM theo điều kiện. Dự án dùng để **tìm quán thật** quanh một điểm (`node["amenity"~"cafe|restaurant|..."]["name"](around:radius,lat,lng)`).
- **OSRM (Open Source Routing Machine)**: tính **tuyến đường** giữa các điểm theo đường phố, trả về quãng đường + thời gian + hình học đường đi (geometry).
- **GitHub Gist API**: lưu một file JSON "bí mật" trên tài khoản GitHub của bạn (cần token quyền `gist`) → dùng để **đồng bộ backup** giữa thiết bị.

**Khái niệm phụ:** **POI (Point of Interest)** = điểm quan tâm (quán, cửa hàng...). `amenity`, `cuisine` là các **tag** trong dữ liệu OSM mô tả loại địa điểm.

---

## 2.6. Lập trình bất đồng bộ & xử lý lỗi mạng

- **`async/await`**: gọi mạng mất thời gian → dùng async để không "đứng hình".
- **`fetch`**: API gọi HTTP của trình duyệt.
- **`fetchWithRetry(url, opts, attempts)`** (tự viết): nếu server trả **429 (Too Many Requests)** hoặc **503**, đợi rồi **thử lại**; hết lượt thì ném lỗi có cờ `rateLimited` để UI báo "bị giới hạn tần suất". Đây là kỹ thuật **resilience** (bền bỉ) khi phụ thuộc dịch vụ dùng chung.
- **`Promise.allSettled`**: chờ nhiều promise xong bất kể cái nào lỗi (dùng khi precache SW và ghi nhiều ảnh).

---

## 2.7. i18n (đa ngôn ngữ)

- `i18n.js` định nghĩa `I18N_STRINGS = { vi: {...}, en: {...} }` (từ điển phẳng theo key như `"filters.title"`).
- Hàm `t(key, params)`: trả chuỗi theo `lang` hiện tại, fallback `vi`, rồi về chính `key`. Hỗ trợ thay `{placeholder}` bằng `params`.
- **UI tĩnh**: HTML gắn thuộc tính `data-i18n` (textContent), `data-i18n-placeholder`, `data-i18n-title`, `data-i18n-aria`; hàm `applyTranslations()` quét và điền.
- **Chuỗi động**: trong JS gọi `t("key")` trực tiếp (toast, nút render...).
- Đổi ngôn ngữ: `toggleLang()` lưu localStorage + `applyTranslations` + dựng lại control + `render`.

---

## 2.8. Playwright & CI/CD (điểm khác biệt của dự án)

- **Playwright**: framework test **end-to-end (E2E)** — điều khiển trình duyệt thật (Chromium...) như người dùng: mở trang, click, điền form, kiểm tra giao diện. Dự án có **47 test**.
  - Test gọi thẳng hàm trong app qua `page.evaluate(() => someFunction())` được vì mọi hàm ở **global scope** (classic script, không module).
  - Các test mạng (Overpass/OSRM/Gist) **mock `window.fetch`** để không gọi mạng thật → test nhanh, ổn định.
- **CI/CD (Continuous Integration / Deployment)**: **GitHub Actions** chạy `npm run check` + `npm test` mỗi khi push lên `main`; job `deploy-pages` **chỉ chạy sau khi test pass** → không bao giờ deploy bản hỏng lên GitHub Pages.

---

## 2.9. Một số thuật ngữ khác

- **Haversine**: công thức tính khoảng cách giữa 2 toạ độ trên mặt cầu Trái Đất (`haversineMeters`) — dùng để tính "gần tôi".
- **Debounce/throttle**: (dự án dùng `render()` gọi lại khi đổi filter; tìm kiếm gắn vào sự kiện `input`).
- **Undo stack**: ngăn xếp ảnh chụp trạng thái (`pushUndoSnapshot`) để **hoàn tác** thao tác thêm/sửa/xóa/đổi tag trong vài giây.
- **Mojibake**: lỗi hiển thị ký tự do sai encoding (ví dụ tiếng Việt thành "Quán" → "QuÃ¡n"); script `check-encoding.js` kiểm tra UTF-8 để tránh.
- **Lightbox**: cửa sổ xem ảnh phóng to, có quản lý focus + đóng bằng Esc.
- **`escapeHtml`/`escapeAttr`**: escape nội dung động (kể cả dữ liệu từ OSM) trước khi nhét vào HTML → chống **XSS**.
- **`normalizePlace`/`isValidPlace`**: chuẩn hóa & kiểm tra dữ liệu quán (điền mặc định, lọc giá trị hợp lệ) — áp cho mọi nguồn (nhập tay, import, OSM, backup).
