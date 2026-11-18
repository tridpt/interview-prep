# 5. Frontend chi tiết (app.js, index.html, sw.js)

> Viết sau khi đọc trực tiếp `frontend/`. Frontend là **vanilla JS thuần**, không framework, không build.

---

## 5.1. `index.html` — Cấu trúc giao diện

Bố cục: một **màn khóa** phủ toàn trang + nội dung chính (4 card) + một modal QR.

| Phần | id chính | Vai trò |
|------|----------|---------|
| Màn khóa | `#lockScreen`, `#lockPassword`, `#lockPassword2`, `#lockBtn`, `#lockHint` | Thiết lập (2 ô mật khẩu) hoặc mở khóa (1 ô) |
| Header | `#themeBtn`, `#toggleHideBtn`, `#lockNowBtn` | Đổi theme / ẩn mã / khóa ngay |
| Sinh mã nhanh | `#quickSecret`, `#quickBtn`, `#quickCode`, `#quickCountdown` | Thử secret bất kỳ, không lưu |
| Lưu tài khoản | `#accName`, `#accSecret`, `#accNote`, `#otpauthUri`, nút quét QR (`#scanCamBtn`/`#scanFileBtn`), `#camWrap`/`#camVideo` | Thêm tài khoản nhiều cách |
| Danh sách | `#search`, `#accountList`, `#emptyMsg` | Hiển thị + tìm |
| Sao lưu | `#exportBtn`, `#importBtnFile`, `#exportCsvBtn`, `#importCsvBtn` + input file ẩn | Backup mã hóa / CSV |
| Modal QR | `#revealModal`, `#revealQr`, `#revealSecret`, `#revealCopy` | Hiện lại QR/secret |

Cuối `<body>` nạp `vendor/jsQR.js` **trước** `app.js`.

---

## 5.2. `app.js` — Logic client (theo nhóm)

### Tiện ích
- **`api(path, options)`**: bọc `fetch` — tự gắn header JSON, parse JSON, **ném `Error(data.detail)` nếu `!res.ok`** → mọi chỗ gọi chỉ cần try/catch.
- `showError/clearError`, `formatCode` (chia "123456" → "123 456" cho dễ đọc), `escapeHtml` (chống XSS khi render tên/ghi chú).

### Sinh mã nhanh
- `doQuick()`: lấy `quickSecret`, gọi `POST /generate`, hiện mã.
- `quickState = {secret, period}` lưu lại để **tự đếm ngược + tự lấy mã mới mỗi chu kỳ**.
- `setInterval(1000)`: tính `remaining = period - (floor(Date.now()/1000) % period)`; khi `remaining === period` (vừa sang chu kỳ) → `refreshQuickCode()` lấy mã mới.

### Thêm / nhập tài khoản
- `addAccount()`: `POST /accounts {name, secret, note}` → reset ô → `loadAccounts()`.
- `importUri()`: `POST /accounts/import-uri {uri}`.

### Quét QR (jsQR, ngay trong trình duyệt)
- **Từ ảnh**: `qrFile` change → vẽ ảnh lên `canvas` ẩn → `getImageData` → `jsQR(data, w, h)` → nếu ra chuỗi → `submitScanned`.
- **Từ camera**: `getUserMedia({video:{facingMode:"environment"}})` → `requestAnimationFrame(scanCameraFrame)` quét từng khung hình; quét được + hợp lệ → lưu & dừng camera; không thì quét tiếp. `stopCamera()` dừng track + hủy rAF.
- `submitScanned(text)`: chỉ chấp nhận chuỗi bắt đầu `otpauth://`, gửi `POST /accounts/import-uri`. **Ảnh/khung hình không bao giờ rời trình duyệt** — chỉ gửi chuỗi otpauth.

### Khóa / mở khóa
- `refreshStatus()`: gọi `GET /status` → `needSetup = !initialized`; nếu `locked` → `showLockScreen()`, ngược lại `hideLockScreen()` + `loadAccounts()`.
- `showLockScreen()`: nếu `needSetup` hiện 2 ô mật khẩu ("Tạo & mở khóa"), ngược lại 1 ô ("Mở khóa").
- `doUnlock()`: nếu setup → kiểm ≥6 ký tự + 2 mật khẩu khớp → `POST /setup`; ngược lại `POST /unlock`. Lỗi → `showError`.
- `lockNowBtn` → `POST /lock` + xóa danh sách + hiện màn khóa. `toggleHideBtn` đảo `hideCodes` (hiện "••• •••").

### Danh sách tài khoản
- `loadAccounts()`: `GET /accounts` → `accounts` → `renderAccounts()`. **Nếu lỗi chứa "khóa" (401 do auto-lock)** → hiện lại màn khóa.
- `renderAccounts()`: lọc theo `search`, dựng mỗi `.account-item` (kéo-thả được): tên, mã (hoặc `••• •••` nếu ẩn), ghi chú, **vòng đếm ngược** (`remaining` + thanh `pct = remaining/period`), nút đổi tên/ghi chú/reveal/xóa. Tài khoản lỗi (secret hỏng) hiện `⚠ error`.
- Bấm mã → `copyCode()` (copy bản không dấu cách) + hiệu ứng "copied".
- **Kéo-thả**: `dragstart` lưu `draggedId`; `drop` → `reorderByDrop(targetId)` sắp xếp lại mảng + `POST /accounts/reorder {ids}` (cập nhật server, không chờ).
- `renameAccount`/`editNote`: dùng `prompt()` rồi `PATCH`.

### Hiện QR / secret
- `revealAccount(id)`: `GET /accounts/{id}/reveal` → nhét `qr_svg` vào modal, hiện secret, nút copy.

### Sao lưu / khôi phục
- **Export mã hóa**: `prompt` mật khẩu (≥6) → `POST /export` → tải file JSON `otp-vault-backup-<ngày>.json`.
- **Import**: đọc file JSON → `prompt` mật khẩu → `POST /import` → báo số mục khôi phục.
- **Export CSV**: `confirm` cảnh báo secret rõ → `fetch /export-csv` → tải `.csv`.
- **Import CSV**: đọc file → `POST /import-csv {csv}` → báo `added/total_rows`.

### Vòng lặp & khởi động
- `setInterval(loadAccounts, 1000)`: **mỗi giây làm mới** mã + đếm ngược khi đã mở khóa.
- Theme: `applyTheme` toggle class `light` trên `<body>`, lưu `localStorage["otpvault-theme"]`.
- `refreshStatus()` lúc tải; đăng ký service worker (PWA).

> 💡 **Điểm bảo mật phía client đáng nói:** (1) `escapeHtml` mọi nội dung động → chống XSS; (2) ảnh QR **không gửi lên server** (giải mã bằng jsQR trong trình duyệt); (3) khi auto-lock, client phát hiện 401 và che màn hình ngay.

---

## 5.3. `sw.js` — Service Worker (PWA)

```js
const CACHE = "otp-vault-v1";
const SHELL = ["/", "/index.html", "/app.js", "/style.css", "/vendor/jsQR.js", "/manifest.json", "/icon.svg"];
// install → cache app shell + skipWaiting
// activate → xóa cache cũ ≠ CACHE + clients.claim
// fetch:
//   if pathname bắt đầu "/api/" → return (KHÔNG xử lý, đi thẳng mạng, KHÔNG cache)
//   chỉ GET → network-first: fetch, lưu bản mới vào cache; lỗi mạng → trả cache (fallback "/")
```
➡️ **Quyết định bảo mật cốt lõi:** service worker **không bao giờ cache `/api/*`** — dữ liệu nhạy cảm (mã, danh sách tài khoản) không được lưu vào cache trình duyệt; và **network-first** đảm bảo luôn lấy dữ liệu tươi (mã 2FA đổi liên tục).

---

## 5.4. Tổng kết quyết định kỹ thuật frontend

| Quyết định | Lý do |
|-----------|-------|
| Vanilla JS, không framework | App nhỏ, một trang; nhẹ, không build |
| `api()` bọc fetch | Xử lý lỗi/JSON nhất quán một chỗ |
| Quét QR bằng jsQR ở client | Ảnh không rời máy; chỉ gửi chuỗi otpauth |
| `setInterval(loadAccounts,1000)` | Mã 2FA đổi liên tục, cần làm mới + đếm ngược |
| Phát hiện 401 → hiện màn khóa | Phản ứng đúng với auto-lock của server |
| SW network-first, không cache /api | Dữ liệu nhạy cảm + cần tươi |
| `escapeHtml` nội dung động | Chống XSS |
