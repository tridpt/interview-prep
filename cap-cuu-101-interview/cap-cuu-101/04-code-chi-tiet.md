# 4. Code chi tiết (giải thích từng file)

File này "mổ xẻ" từng file mã. Đọc kèm code thật trong `cap-cuu-101/`.

---

## 4.1. `index.html` — Khung & nội dung tĩnh

**Nhiệm vụ:** chứa toàn bộ DOM của app dưới dạng 10 màn `<section class="screen">`, các overlay, và metadata.

**Điểm chính:**
- Phần `<head>` khai báo đầy đủ **PWA + SEO + link preview**: `manifest`, `theme-color`, `apple-touch-icon`, các meta `apple-mobile-web-app-*` (để cài như app trên iOS), và **Open Graph + Twitter Card** (ảnh `og.png` 1200×630) để link dán lên Zalo/Facebook hiện đẹp.
- `viewport` đặt `user-scalable=no` — giao diện game cố định, không cho zoom.
- Các nút điều hướng dùng thuộc tính `data-goto="screen-xxx"` → app.js gắn sự kiện chung cho mọi `[data-goto]`.
- Các phần tử cần dịch gắn `data-i18n="key"` → `applyLang()` quét và điền text.
- Nạp `sound.js` **trước** `app.js` (để `window.Sfx` sẵn sàng).

---

## 4.2. `sound.js` — Module âm thanh (Web Audio)

**Nhiệm vụ:** lộ ra `window.Sfx`, tự tổng hợp mọi âm thanh, quản lý cài đặt sound/music/vibrate. Bọc trong IIFE.

**Các phần quan trọng:**
- `loadSettings()/saveSettings()`: đọc/ghi `capcuu101_settings` (mặc định `{sound:true, music:false, vibrate:true}`), bọc `try/catch` phòng JSON hỏng.
- `ensure()`: tạo `AudioContext` lần đầu (lazy), tạo một `master` GainNode (âm lượng tổng). Nếu context bị `suspended` thì `resume()`.
- `tone(freq, start, dur, type, vol)`: tạo **một nốt** — một `OscillatorNode` (sóng) + `GainNode` (bao âm lượng: tăng nhanh rồi tắt dần bằng `exponentialRampToValueAtTime`).
- `play(seq)`: phát một chuỗi nốt theo mốc thời gian; **bỏ qua nếu tắt âm** (`!settings.sound`).
- Các hiệu ứng: `tick()` (tách metronome), `correct()` (ting lên), `wrong()` (sai), `logout()` ("đăng xuất khỏi cuộc đời" — kèn sawtooth tụt dốc buồn), `win()` (fanfare đi lên), `blip()`.
- `startMusic()/stopMusic()`: nhạc nền 8-bit bằng `setInterval` phát giai điệu + bass vòng lặp.
- `vibrate(pattern)`: gọi `navigator.vibrate` nếu bật và thiết bị hỗ trợ.
- `unlock()`: gọi `ensure()` trong lần chạm đầu để vượt **autoplay policy** của trình duyệt.

> 💡 **Điểm ăn điểm:** giải thích được vì sao **tự tổng hợp âm thanh** thay vì dùng file: app shell nhẹ, offline hoàn hảo, không cần cache media. Và biết về **autoplay policy** (phải có user gesture mới phát được tiếng) là chi tiết "thực chiến".

---

## 4.3. `app.js` — Bộ não (IIFE ~2.980 dòng)

Toàn bộ bọc trong `(() => { "use strict"; ... })()`. Các nhóm chức năng:

### i18n
- `UI`: từ điển `{ key: {vi, en} }`.
- `t(v)`: trả chuỗi theo `LANG` nếu `v` là object song ngữ.
- `applyLang()`: quét mọi `[data-i18n]` đặt `textContent`; bật class `body.lang-en` cho khối `.only-vi/.only-en`.
- `setLang()`: lưu localStorage + `applyLang()` + render lại màn đang mở.

### Router
- `go(id)`: dừng hoạt động màn cũ, đổi `.active`, cập nhật `aria-hidden`, gọi render tương ứng. Một số tham chiếu (`emergStop`, `emergRender`) được gán động để `go` gọi ngược vào module con.

### Engine game (xem chi tiết ở file 03 mục 3.5)
- `startGame(lv)` → tutorial lần đầu → `runGame(lv)` → `initRhythm/initThrust/initHose/initPress/initQuiz`.
- Mỗi init đặt `gameState.cleanup`; `stopGame()` gọi cleanup khi rời màn (chống rò rỉ).
- `showResult(lv, success, stars, extra)`: overlay kết quả gắn vào **`#screen-game`** (phủ toàn màn, cuộn được), hiện sao + bài học + huy hiệu mới + nút chia sẻ/chơi lại/danh sách.
- `startSurvival/initSurvival`: hỏi liên tục, đồng hồ đếm ngược bằng `requestAnimationFrame`, 1 sai là hết, lưu `best.survival`.

### Panic Mode & TTS
- `openPanic(key)`: chọn dataset theo `LANG`, `renderPanicStep()` cập nhật text + nút + `speakStep()`.
- `refreshVoices()`: nạp `getVoices()` (async, lắng `onvoiceschanged`).
- `pickSpeakVoice()`: ưu tiên giọng đã chọn → giọng theo `LANG` → `"off"`.
- `hasViVoice()`: kiểm tra để cảnh báo khi thiếu giọng Việt.
- Metronome Panic (CPR): `setInterval` 600ms, vòng ring nảy + rung + đếm 1-30.
- `formatStepText()/renderPanicStep()`: thay "một một năm" → "115", "năm centimet" → "5cm" khi hiển thị.

### Công cụ khẩn cấp & Cẩm nang
- `initEmergency()` (IIFE): máy đếm nhịp CPR độc lập (110 BPM), thẻ y tế (escape HTML chống XSS, nút `tel:` lọc số còn `[\d+]`), gán `emergStop`/`emergRender`.
- `renderHandbook()`: dựng accordion từ `PANIC(_EN)` theo `HANDBOOK_ORDER`.

### Thành tích, streak, prefs
- `loadStats()/saveStats()` (`capcuu101_stats`); `recordWin(id)` tăng wins + cập nhật streak theo ngày (`todayStr`, `daysBetween`); `saveBest(id, val)`.
- `checkBadges()`: trả về huy hiệu mới mở khoá.
- `PREFS` (`capcuu101_prefs`): contrast/large/reduce/serious → `applyPrefs()` bật class trên `<body>`.

---

## 4.4. `sw.js` — Service Worker (offline)

**Nhiệm vụ:** precache app shell để chạy offline 100%.

```javascript
const CACHE_VERSION = "capcuu101-v17";
const APP_SHELL = ["./", "./index.html", "./style.css", "./sound.js",
                   "./app.js", "./manifest.json", ...icons, ...screenshots];
```

- **install**: `cache.add` từng file bằng `Promise.allSettled` (một file lỗi không làm hỏng cả cài đặt) → `skipWaiting()` để SW mới kích hoạt ngay.
- **activate**: xoá mọi cache không bắt đầu bằng `CACHE_VERSION` hiện tại → `clients.claim()` để kiểm soát trang ngay.
- **fetch**:
  - Chỉ xử lý `GET`.
  - `request.mode === "navigate"` (điều hướng trang) → `networkFirstPage`: thử mạng, lưu vào cache, rớt mạng thì trả `./` hoặc `./index.html` từ cache.
  - Còn lại → `staleWhileRevalidate`: trả cache ngay, đồng thời fetch ngầm cập nhật cache.

> 💡 **Câu hỏi hay gặp: "Vì sao 2 chiến lược khác nhau?"** Trang HTML cần **mới** (network-first) để người dùng không kẹt bản cũ khi có mạng; còn asset (css/js/ảnh) ưu tiên **tốc độ** (stale-while-revalidate) vì chúng ít đổi và được version-hoá qua `CACHE_VERSION`.

---

## 4.5. `manifest.json` — Khai báo PWA

- `name`, `short_name`, `description`, `icons` (192/512 + SVG, `purpose: "any maskable"`).
- `display: "standalone"` (chạy như app, không thanh địa chỉ), `orientation: "portrait"`.
- `theme_color`/`background_color` cho splash & thanh trạng thái.
- `shortcuts`: 2 lối tắt — "Chế độ Hoảng loạn" (`?panic=1`) và "Công cụ khẩn cấp" (`?tools=1`).
- `screenshots`: ảnh cho prompt "Install" của trình duyệt hiển thị đẹp hơn.

---

## 4.6. Các script `gen_*.py` (chỉ dùng lúc dev)

`gen_icons.py`, `gen_screens.py`, `gen_og.py`, `gen_gif.py` dùng **Pillow (Python)** sinh ra icon PNG, ảnh screenshot, ảnh Open Graph, GIF demo. **Chạy một lần** lúc dev, không nằm trong runtime của app. Đây là lý do duy nhất dự án "đụng" tới Python.

---

## 4.7. Tổng kết các quyết định kỹ thuật

| Quyết định | Lý do |
|-----------|-------|
| Vanilla JS, không framework/thư viện | Offline-first, nhẹ tuyệt đối, không bề mặt tấn công chuỗi cung ứng |
| Tự tổng hợp âm thanh (Web Audio) | Không file media → app shell nhẹ, offline hoàn hảo |
| Service worker + cache versioning | Chạy offline 100% (sống còn cho app cấp cứu) |
| 2 chiến lược cache | Trang cần mới (network-first), asset cần nhanh (SWR) |
| `gameState.cleanup` cho mỗi game | Chống rò rỉ tài nguyên khi không có framework |
| Một nguồn dữ liệu (PANIC dùng chung) | DRY, an toàn nội dung y tế |
| i18n tự viết | Đủ dùng, không thêm phụ thuộc |
| Mọi dữ liệu ở localStorage | Riêng tư, không backend |
| Escape HTML thẻ y tế | Chống XSS |
| `?panic=1` / `?tools=1` | Mở thẳng tính năng khẩn cấp qua shortcut PWA |
