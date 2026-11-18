# 1. Tổng quan dự án Quán quen (Taste Map)

## 1.1. Quán quen là gì?

Quán quen là một **web app dạng bản đồ** để bạn lưu lại những quán ăn/cafe "ruột" của mình và quản lý chúng **theo gu cá nhân**. Khác với Google Maps (chấm điểm chung chung của cộng đồng), Quán quen cho bạn ghi lại cảm nhận **của chính bạn**: quán này *ngon*, quán kia *yên tĩnh*, quán nọ *có ổ cắm hợp làm việc* hay *hợp đi date*. Rồi app dùng dữ liệu đó để gợi ý "hôm nay đi đâu", chỉ đường, lập lịch trình và khám phá quán mới quanh bạn.

### Điểm cốt lõi cần nhớ
- **Bản đồ cá nhân hóa**: chấm điểm 5 tiêu chí riêng (ngon/yên tĩnh/ổ cắm/date/làm việc).
- **Không backend, riêng tư tuyệt đối**: dữ liệu nằm trong trình duyệt (localStorage + IndexedDB), không tài khoản, không server.
- **PWA chạy offline**, song ngữ Việt/Anh.
- **Dùng dữ liệu bản đồ thật**: OpenStreetMap (nền + khám phá quán), Nominatim (tìm địa chỉ), OSRM (định tuyến).
- **Có kiểm thử tự động + CI/CD** — điểm khác biệt lớn so với nhiều dự án nhỏ.

## 1.2. Dự án giải quyết vấn đề gì?

1. **Google Maps không nhớ "gu" của bạn** → Quán quen lưu cảm nhận cá nhân theo nhiều tiêu chí.
2. **Phân vân "hôm nay đi đâu"** → gợi ý theo mood (làm việc/date/yên tĩnh/ăn nhanh/ăn ngon/gần tôi/mới chưa thử).
3. **Đi nhiều quán một buổi** → lập lịch trình, vẽ tuyến đường thật + thời gian, mở chỉ đường đa điểm.
4. **Muốn riêng tư** → mọi dữ liệu ở máy bạn, không gửi đi đâu (trừ khi tự bật đồng bộ Gist).
5. **Trình diễn năng lực kỹ thuật**: bản đồ tương tác, lưu trữ nâng cao (IndexedDB), mã hóa, tích hợp nhiều dịch vụ ngoài, **và kỹ luật kỹ thuật** (test tự động + CI/CD) — đây là phần "ăn điểm".

> 💡 **Mẹo phỏng vấn:** dự án này nổi bật vì **độ chín về kỹ thuật**: không chỉ làm tính năng mà còn có 47 test Playwright, kiểm tra encoding, và pipeline CI/CD chỉ deploy khi mọi thứ pass. Hãy nhấn vào điều này — nó cho thấy bạn quan tâm chất lượng, không chỉ "chạy được".

## 1.3. Các tính năng chính

| Nhóm | Tính năng |
|------|-----------|
| **Bản đồ** | Leaflet + OSM tiles, marker theo điểm hợp gu, gom cụm (markercluster), lớp heatmap mật độ |
| **Khám phá quán thật** | "Khám phá khu này" / "Random gần tôi" quét OpenStreetMap (Overpass), lọc theo loại, bán kính 1-10km |
| **Định tuyến** | Lịch trình nhiều điểm, vẽ tuyến thật qua OSRM (fallback đường chim bay), mở chỉ đường Google/Apple Maps |
| **Lọc & tìm kiếm** | Theo loại/gu/tag/khu vực/giá/khoảng cách/đang mở cửa; sắp xếp 5 kiểu; tìm tự do |
| **CRUD quán** | Thêm/sửa/xóa, đánh giá 5 tiêu chí, giờ mở cửa, lịch sử ghé, quán ruột, **hoàn tác (undo)** |
| **Ảnh** | Nhiều ảnh, nén, phân album (món/không gian/menu/khác), lưu trong **IndexedDB**, lightbox, dọn ảnh mồ côi |
| **Tổ chức** | Bộ sưu tập tùy chỉnh, quản lý tag (đổi tên/xóa hàng loạt), loại quán tùy chỉnh |
| **Gợi ý & thống kê** | "Hôm nay đi đâu?" theo mood, nhắc quán ruột lâu chưa ghé, thống kê lần ghé + chi tiêu 6 tháng |
| **Nhập/xuất** | Import Google Maps link, CSV in/out, backup JSON thường + **mã hóa (AES-GCM)**, đồng bộ GitHub Gist |
| **Giao diện** | Dark/light mode, song ngữ VI/EN, responsive, toast có nút hành động |
| **PWA & offline** | manifest + service worker, cache app shell + tile, share_target nhận link từ điện thoại |
| **Chất lượng** | 47 test Playwright, kiểm encoding UTF-8, CI/CD GitHub Actions |

## 1.4. Công nghệ sử dụng (tech stack)

**Cốt lõi (client):**
- **HTML5 + CSS3 + JavaScript thuần (ES2020)** — không framework, **không bundler/build step**.
- **Leaflet 1.9.4** — thư viện bản đồ; **Leaflet.markercluster 1.5.3** — gom cụm marker; **leaflet.heat 0.2.0** — heatmap.
- **Lucide 1.17.0** — bộ icon (gọi `lucide.createIcons()`).
- Các thư viện nạp qua **CDN unpkg, ghim phiên bản cố định** (không dùng `@latest`).

**Lưu trữ:**
- **localStorage** — dữ liệu quán (dạng text), cấu hình, bộ sưu tập, lịch trình.
- **IndexedDB** — ảnh quán (base64) tách riêng để localStorage không bị đầy.
- **Web Crypto API** — backup mã hóa (PBKDF2-SHA-256 250k vòng + AES-GCM).

**Dịch vụ web ngoài (miễn phí):**
- **OpenStreetMap tiles** (nền bản đồ), **Nominatim** (geocode/reverse), **Overpass API** (tìm quán thật), **OSRM** (định tuyến), **GitHub Gist API** (đồng bộ tùy chọn).

**Dev / Test / CI:**
- **Node + Playwright** (47 test E2E), **http-server** (chạy local), **GitHub Actions** (CI: check + test → deploy GitHub Pages).
- Script `check-encoding.js` kiểm tra UTF-8 tránh mojibake.

> 💡 **Câu hỏi cài bẫy: "Sao không framework cho app phức tạp thế này?"** Trả lời: "App phức tạp về *tính năng* nhưng kiến trúc state đủ đơn giản để quản bằng vanilla (một hàm `render()` đọc state vẽ lại UI). Không bundler nghĩa là **deploy = đẩy file tĩnh**, dễ host trên GitHub Pages, và **test Playwright gọi thẳng được hàm trong app** qua `page.evaluate` vì mọi hàm ở global scope. Đổi lại em tự quản state và DOM — bù lại bằng 47 test tự động để giữ chất lượng."

## 1.5. Cách chạy

```powershell
cd quan-quen-map
npm run serve        # http-server tại http://127.0.0.1:5178
# hoặc: python -m http.server 5178
```

Kiểm tra & test:
```powershell
npm run check        # check encoding UTF-8 + node --check app.js/i18n.js/sw.js + parse manifest
npm install          # lần đầu
npx playwright install
npm test             # 47 test Playwright (tự khởi động server)
```

> Service worker chỉ chạy qua `http(s)://` (localhost hoặc GitHub Pages), không chạy với `file://`.

## 1.6. Cấu trúc thư mục

```
quan-quen-map/
├── index.html          # Khung HTML + toàn bộ DOM tĩnh; nạp i18n.js → app.js
├── app.js              # Toàn bộ logic (~4.300 dòng)
├── i18n.js             # Từ điển dịch VI/EN (I18N_STRINGS)
├── style.css           # CSS, biến theme, dark mode, responsive (~2.710 dòng)
├── sw.js               # Service worker (cache app shell + tile + runtime)
├── manifest.json       # PWA manifest (icon, share_target)
├── scripts/            # check-encoding.js, screenshots.js, gif.js (dev)
├── tests/app.spec.js   # 47 test Playwright
├── playwright.config.js
├── package.json        # scripts: serve / test / check
└── .github/workflows/ci.yml   # CI: check + test → deploy Pages
```

Quy mô: ~8.800 dòng (app.js ~4.718, style.css ~2.710, i18n.js ~688, index.html ~574, sw.js ~148).

> ⚠️ **Lưu ý dịch vụ ngoài:** Overpass/OSRM/Nominatim là dịch vụ dùng chung miễn phí, có thể chậm hoặc giới hạn tần suất (đã có `fetchWithRetry` thử lại khi 429/503 và thông báo cho người dùng). Quán hiển thị mặc định là quán **đã lưu**; quán thật chỉ xuất hiện khi bấm "Khám phá".
