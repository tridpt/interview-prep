# 3. Kiến trúc hệ thống

## 3.1. Sơ đồ tổng thể

```
┌──────────────────────────── TRÌNH DUYỆT ────────────────────────────┐
│                                                                       │
│  index.html ──▶ DOM tĩnh: 10 <section class="screen"> + data-i18n     │
│  sound.js   ──▶ window.Sfx (Web Audio: tick/correct/wrong/win/bgm+rung)│
│  app.js     ──▶ IIFE: i18n (UI dict + t()) · router go() · dữ liệu     │
│                  nội dung · engine game · Panic Mode (TTS) · công cụ   │
│  style.css  ──▶ theme tối, chuyển màn, a11y, responsive               │
│        │              │                │                              │
│        ▼              ▼                ▼                              │
│  localStorage    Web Speech (TTS)   Các Web API:                      │
│  (sao, stats,    + voice picker     Vibration, Notifications,         │
│   prefs, thẻ y tế)                  DeviceOrientation, Web Share,     │
│                                     Canvas, Notifications              │
│                                                                       │
│                          ▼                                            │
│                   sw.js (Service Worker)                              │
│                   precache app shell → CHẠY OFFLINE 100%              │
└───────────────────────────────────────────────────────────────────────┘

      ⚠️ KHÔNG có request mạng ra ngoài (trừ link "Nguồn tham khảo"
         chỉ mở tab mới khi người dùng tự bấm).
```

**Đặc điểm kiến trúc cốt lõi:** đây là một app **hoàn toàn phía client (client-side only)**, **không có backend**. Mọi logic, dữ liệu nội dung, và trạng thái đều nằm trong trình duyệt. Đây là điểm khác biệt lớn nhất so với các dự án có server.

## 3.2. Phân lớp & trách nhiệm các file

| File | Vai trò | Trách nhiệm |
|------|---------|-------------|
| `index.html` | Khung & nội dung tĩnh | 10 màn dạng `<section class="screen">` + overlay; gắn `data-i18n` cho i18n |
| `sound.js` | Module âm thanh | Lộ `window.Sfx`; tổng hợp âm thanh Web Audio + cài đặt; nạp **trước** app.js |
| `app.js` | Bộ não | i18n, router, dữ liệu nội dung, engine game, Panic, công cụ, thành tích, đăng ký SW |
| `style.css` | Giao diện | Theme, animation, accessibility, responsive |
| `sw.js` | Offline | Service worker: precache + chiến lược cache |
| `manifest.json` | PWA | Khai báo app, icon, shortcut, screenshot |

> 💡 **Điểm đáng nói:** Toàn bộ logic gói trong **một file `app.js` ~2.980 dòng**, bọc trong một IIFE. Đây vừa là điểm mạnh (đơn giản, không build, dễ cache) vừa là **giới hạn đã biết** (file lớn, nếu mở rộng nhiều nên tách module). Biết tự nhận ra điều này là một điểm cộng về tư duy kỹ thuật.

## 3.3. Luồng khởi động (init flow) — học kỹ

Khi app mở, `app.js` chạy theo trình tự (ở cuối file):

```
1. Định nghĩa UI (từ điển i18n) + hàm t() (dịch).
2. Thu thập các .screen, gắn sự kiện cho mọi [data-goto] (nút điều hướng).
3. Định nghĩa dữ liệu & hàm: stats/badges, LEVELS, render, engine game,
   QUIZZES(_EN), PANIC(_EN), TTS, metronome, công cụ khẩn cấp, handbook, settings.
4. applyLang() + applyPrefs() — áp ngôn ngữ & tùy chọn (tương phản/chữ lớn...) lên <body>.
5. Đặt aria-hidden cho mọi màn trừ màn hiện tại (accessibility).
6. Đọc query string:
      ?panic=1 → mở thẳng Chế độ Hoảng loạn
      ?tools=1 → mở thẳng Công cụ khẩn cấp
      mặc định → Home
7. maybeRemind() sau 1.8s — nhắc ôn tập nếu chuỗi ngày đang chờ nối.
8. Đăng ký service worker (khi 'load').
9. hideSplash() — ẩn màn splash sau ~1.4s.
```

**Vì sao đọc query string?** Để 2 shortcut PWA (`?panic=1`, `?tools=1`) mở thẳng vào đúng tính năng — quan trọng cho tình huống khẩn cấp, mở app là vào ngay hướng dẫn.

## 3.4. Router đơn giản — hàm `go(id)`

Đây là "bộ điều hướng" client-side, không tải lại trang:

```
go(id):
  1. Dừng game/panic/metronome của màn cũ (tránh rò rỉ tài nguyên).
  2. Đổi class .active sang màn mới.
  3. Cập nhật aria-hidden cho các màn.
  4. Gọi hàm render tương ứng (renderLevels / renderAchievements /
     renderHandbook / emergRender).
```

**10 màn:** `screen-home`, `screen-levels`, `screen-game`, `screen-panic`, `screen-about`, `screen-sources`, `screen-achievements`, `screen-emergency`, `screen-medcard`, `screen-handbook`. Cộng các overlay: `#splash`, `#tutorial`, `#settings-modal`.

## 3.5. Engine mini-game (5 kiểu game)

Tất cả game dùng chung 3 phần tử DOM (`#game-stage`, `#game-instruction`, `#game-controls`) và một biến `gameState = { cleanup() }`.

**Mẫu thiết kế quan trọng — `cleanup()`:** mỗi game khi khởi tạo sẽ đặt một hàm `gameState.cleanup`. Khi rời màn, `stopGame()` gọi `cleanup()` để **dọn dẹp** (xoá listener, clear interval, dừng cảm biến). Đây là cách chống **rò rỉ bộ nhớ/tài nguyên** khi không có framework quản lý vòng đời.

| `type` | Hàm | Cơ chế |
|--------|-----|--------|
| `rhythm` | `initRhythm` | Chạm đúng nhịp (`interval = 60000/BPM`), tính combo + độ chính xác, tiếng tách + rung; chơi được bằng Space/Enter |
| `thrust` | `initThrust` | DeviceOrientation (nghiêng) đẩy bụng; fallback nhấn-giữ-thả nếu không cảm biến |
| `hose` | `initHose` | Pointer rê "vòi nước" đè vết bỏng đủ lâu; vết bỏng định kỳ "lan" |
| `press` | `initPress` | Bước 1 kéo gạc vào vòng tròn; bước 2 giữ lực trong vùng xanh dao động |
| `quiz` | `initQuiz` | Trắc nghiệm, sai trừ 40 HP, hết HP thì thua |

Luồng: `startGame(lv)` → hiện tutorial lần đầu → `runGame(lv)` điều phối theo `type` → `showResult()` (chấm sao, bài học, huy hiệu mới, nút chia sẻ/chơi lại).

## 3.6. Mô hình dữ liệu (data model)

**Dữ liệu nội dung** (hằng trong `app.js`):
- `LEVELS`: mảng 17 màn — `{ id, emoji, type, char, title{}, desc{}, lesson{} }`.
- `QUIZZES` / `QUIZZES_EN`: `{ [id]: [{ q, options:[{t,ok}], wrongMsg }] }`, mỗi câu đúng 1 đáp án.
- `PANIC` / `PANIC_EN`: `{ [id]: { title, metronome?, steps:[...] } }`, 17 tình huống.
- `TUTORIALS`, `BADGES` (9 huy hiệu với hàm `test(stats, stars)`), `HANDBOOK_ORDER`.

**Dữ liệu người dùng** (localStorage):

| Khóa | Nội dung |
|------|----------|
| `capcuu101_stars` | Số sao mỗi màn |
| `capcuu101_stats` | wins, totalWins, best (combo/survival), streak, badges |
| `capcuu101_settings` | sound / music / vibrate (quản bởi `sound.js`) |
| `capcuu101_prefs` | contrast / large / reduce / serious |
| `capcuu101_lang` | ngôn ngữ (vi/en) |
| `capcuu101_voice` | giọng đọc đã chọn |
| `capcuu101_tut` | đã xem hướng dẫn kiểu game nào |
| `capcuu101_reminder` | bật nhắc ôn tập |
| `capcuu101_medcard` | thẻ y tế cá nhân |

> 💡 **Một nguồn dữ liệu duy nhất:** Cẩm nang tra cứu **tái dùng đúng** dữ liệu `PANIC`/`PANIC_EN` của Chế độ Hoảng loạn — không trùng lặp nội dung. Đây là nguyên tắc DRY (Don't Repeat Yourself), đặc biệt quan trọng với app y tế: sửa một chỗ, đúng mọi nơi.

## 3.7. Tầng offline — Service Worker (`sw.js`)

```
install  → precache APP_SHELL (mọi file cục bộ) vào Cache API → skipWaiting()
activate → xoá mọi cache không thuộc CACHE_VERSION hiện tại → clients.claim()
fetch    → điều hướng trang: network-first (rớt mạng → trả index từ cache)
           tài nguyên khác: stale-while-revalidate
```

Mỗi lần đổi asset phải **bump `CACHE_VERSION`** (vd v17 → v18) để client nhận bản mới. `Promise.allSettled` khi precache đảm bảo một file lỗi không làm hỏng cả quá trình cài.

## 3.8. Vì sao kiến trúc này hợp lý cho một app cấp cứu?

- **Offline-first**: lúc khẩn cấp có thể mất mạng → app vẫn chạy.
- **Nhẹ tuyệt đối**: không framework/thư viện → tải nhanh, cache gọn, ít rủi ro.
- **Riêng tư**: không backend, mọi dữ liệu ở máy người dùng → không lo lộ dữ liệu y tế cá nhân.
- **Dễ deploy & bền**: web tĩnh trên GitHub Pages, không server để bảo trì hay sập.
