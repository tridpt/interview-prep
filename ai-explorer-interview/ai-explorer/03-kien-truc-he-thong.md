# 3. Kiến trúc hệ thống

File này mô tả **cách các mảnh ghép nối với nhau**: luồng khởi động, router, mảng `ROOMS`, hệ thống lộ trình, và cách thêm một phòng mới. Nắm chắc phần này là bạn "kể được câu chuyện kỹ thuật" của dự án.

---

## 3.1. Bức tranh tổng thể

AI Explorer là một **Single-Page App phía client, không backend**. Mọi thứ chạy trong trình duyệt:

```
index.html  (nạp app.js dưới dạng <script type="module">)
   │
   ├─ app.js  ── bộ điều phối trung tâm
   │     ├─ kết hợp metadata (rooms-meta.js) với lazy loader (room-loaders.js)
   │     ├─ định nghĩa mảng ROOMS (mạch kể chuyện)
   │     ├─ THEME / HINTS / SEARCH_KEYWORDS
   │     ├─ router: hashchange → route()
   │     ├─ toolbar (ngôn ngữ / âm thanh / tìm / trình chiếu)
   │     └─ glue: i18n, store, sound, analytics, roomstate, roomquiz
   │
   ├─ i18n.js / store.js / roomstate.js / roomquiz.js / tracks.js / sound.js  (hệ con)
   ├─ pwa.js → đăng ký sw.js (service worker offline)
   └─ rooms/*.js  ── mỗi phòng export render(root), tự lo nội dung + tương tác
```

Nguyên tắc thiết kế: **`app.js` lo khung, mỗi phòng lo ruột.** `app.js` dựng tiêu đề, thanh tiến trình, nút điều hướng, chèn quiz nhỏ; còn nội dung tương tác của phòng nằm gọn trong module phòng.

`rooms-meta.js` giữ metadata nhẹ để cả `app.js` và `build-seo.mjs` dùng chung; `room-loaders.js` dùng dynamic `import()` để chỉ nạp code phòng khi truy cập. `build-seo.mjs` sinh trang tĩnh `vi/`, `en/` và sitemap; CI chạy `test:all`, prerender và Lighthouse trước deploy. `privacy.html` giải thích analytics tùy chọn.

---

## 3.2. Luồng khởi động

Cuối `app.js`:
```js
buildToolbar();   // dựng thanh công cụ (ngôn ngữ/âm thanh/tìm/trình chiếu)
initAnalytics();  // khởi tạo thống kê ẩn danh (tuỳ chọn)
route();          // render phòng theo hash hiện tại
```
Song song, `index.html` nạp `pwa.js` → đăng ký service worker để bật khả năng offline và theo dõi bản cập nhật.

---

## 3.3. Mảng `ROOMS` — trái tim của điều hướng

`ROOMS` là một mảng object, **thứ tự = mạch kể chuyện của hành trình**. Mỗi phần tử:

```js
{
  id: "reasoning", icon: "💭", num: "25",
  title:   { vi: "AI biết suy nghĩ từng bước", en: "AI that thinks step by step" },
  question:{ vi: "Vì sao 'nghĩ' giúp AI trả lời đúng hơn?", en: "…" },
  blurb:   { vi: "…", en: "…" },
  render:  ROOM_LOADERS["reasoning"], // hàm async trả về render(root)
}
```

`ROOMS` được dùng ở khắp nơi:
- **Router** tìm phòng theo `id`.
- **Thanh tiến trình** (`renderProgress`) tạo một chấm cho mỗi phòng.
- **Nút Trước/Sau** lấy `ROOMS[idx-1]` / `ROOMS[idx+1]`.
- **Trang chủ** (`home.js`) render lưới thẻ phòng.
- **Ô tìm kiếm** lọc trên `ROOMS` + `SEARCH_KEYWORDS`.
- **Smoke test** đọc trực tiếp `ROOMS` từ mã nguồn để duyệt mọi phòng.

> 💡 Thứ tự và nội dung mô tả phòng ở `rooms-meta.js`, còn `room-loaders.js` ánh xạ ID sang import động. Prerender SEO dùng cùng metadata; thêm phòng cần cập nhật cả hai file và asset offline.

---

## 3.4. `route()` — dựng một phòng như thế nào

Khi hash đổi, `route()` chạy tuần tự:

1. Phát sự kiện `"roomleave"` để phòng cũ dọn dẹp (clear timer, tắt webcam, dừng animation).
2. Xoá nội dung cũ, cuộn lên đầu, chạy lại animation vào.
3. `applyTheme(id)` — đặt các biến CSS màu chủ đề theo phòng (xem 3.5).
4. `trackView(...)` — ghi lượt xem (analytics ẩn danh).
5. Nếu là `home` → `renderHome(...)` rồi dừng.
6. Ngược lại, tìm phòng trong `ROOMS`; nếu không có → về home.
7. Đánh dấu đã ghé (`markVisited`) + cập nhật thanh tiến trình.
8. Dựng **khung sườn chung**: tiêu đề phòng (icon, câu hỏi, tên) + nút Chia sẻ.
9. `await room.render()` nạp module, kiểm tra route chưa đổi, rồi gọi hàm trả về với `body`; lỗi tải có thông báo.
10. Nếu phòng có quiz nhỏ → `renderMicroQuiz(body, id)`.
11. Hiện gợi ý onboarding (`showHint`).
12. Dựng nút điều hướng Trước / Sau.

```js
const room = ROOMS[idx];
visited.add(room.id);
markVisited(room.id);
renderProgress(room.id);
// … dựng head …
const renderRoom = await room.render();
if (generation !== routeGeneration || currentRoute() !== room.id) return;
renderRoom(body);
if (room.id !== "summary" && hasMicroQuiz(room.id)) renderMicroQuiz(body, room.id);
showHint(room.id);
```

---

## 3.5. Chủ đề màu & tương phản WCAG (`applyTheme`)

Mỗi phòng có một bộ màu trong `THEME[id] = [accent, accent2, rgb]`. `applyTheme` đặt các biến CSS:

```js
function applyTheme(id) {
  const [a, a2, rgb] = THEME[id] || THEME.home;
  const r = document.documentElement.style;
  r.setProperty("--accent", a);
  r.setProperty("--accent-2", a2);
  r.setProperty("--accent-rgb", rgb);
  r.setProperty("--accent-ink", darkenForText(a)); // màu ĐẬM chỉ dùng cho chữ
}
```

Điểm đáng nói: `darkenForText()` **làm đậm dần** màu chủ đề (nhân 0.9 mỗi vòng) cho tới khi đạt tương phản ≥ 4.5:1 với nền sáng (dùng công thức luminance/contrast của WCAG). Nhờ vậy chữ theo tông từng phòng vẫn đọc được mà khối màu không đổi — một chi tiết accessibility "ăn điểm".

---

## 3.6. Hệ thống lộ trình (`tracks.js`)

Để người mới không choáng với 26 phòng, có 3 lộ trình:

| Lộ trình | Nội dung | Phòng |
|----------|----------|-------|
| `beginner` 🌱 | 5 phòng cốt lõi, ~10 phút | teachable, neural-net, tokenizer, next-token, summary |
| `full` 🧭 | Toàn bộ hành trình | tất cả (rooms = `null`) |
| `dev` ⌨️ | Cho lập trình viên | tokenizer, embeddings, attention, next-token, rag, finetune, agents, multimodal, **reasoning**, chatbot, summary |

`trackRoomIds(trackKey, allRooms)` trả danh sách id theo lộ trình (fallback: tất cả). Trang chủ (`home.js`) dùng nó để **làm mờ** phòng ngoài lộ trình và đánh dấu "Trong lộ trình" cho phòng thuộc lộ trình. Nút "Bắt đầu" nhảy tới phòng chưa ghé đầu tiên *trong lộ trình*.

> 💡 Lưu ý: phòng **reasoning** nằm trong lộ trình *Cho lập trình viên* — một cách nhấn mạnh nó là chủ đề nâng cao, thời sự.

---

## 3.7. Deep-link & chia sẻ (kết hợp roomstate.js)

- Phòng ghi trạng thái bằng `setParams({ p: idx })` → URL thành `#/reasoning?p=2` (dùng `replaceState`, không render lại).
- Khi mở phòng, phòng đọc `getParam("p")` để khôi phục đúng trạng thái.
- Nút "Chia sẻ" gọi `shareRoom(room)`: dùng `navigator.share` trên điện thoại, fallback copy link (kèm trạng thái hiện tại) vào clipboard trên desktop.

---

## 3.8. Cách thêm một phòng mới (checklist đầy đủ)

Đây là câu hỏi phỏng vấn kinh điển về khả năng mở rộng. Các bước:

1. **Tạo `rooms/ten-phong.js`** export một hàm `render(root)` (theo quy ước đặt tên `roomTenPhong`). Nếu phòng có animation/timer, nhớ lắng `roomleave` để dọn dẹp.
2. **Thêm import + một mục vào mảng `ROOMS`** trong `app.js` (đặt đúng vị trí trong mạch kể chuyện; điền `id/icon/num/title/question/blurb` song ngữ và `render`).
3. **Thêm màu chủ đề** vào `THEME[id]`.
4. **Thêm gợi ý onboarding** vào `HINTS[id]` (tuỳ chọn).
5. **Thêm từ khoá tìm kiếm** vào `SEARCH_KEYWORDS[id]`.
6. **Thêm quiz nhỏ** vào `BANK[id]` trong `roomquiz.js` (tuỳ chọn).
7. **Thêm đường dẫn file vào mảng `ASSETS`** trong `sw.js` **và bump `CACHE`** (vd v22 → v23) — nếu quên, `check-assets.mjs` sẽ báo lỗi và chặn deploy.
8. (Tuỳ chọn) thêm vào lộ trình trong `tracks.js`.

Khung sườn (tiêu đề, thanh tiến trình, nút điều hướng, quiz nhỏ) được `app.js` xử lý tự động, và **smoke test tự đọc `ROOMS`** nên phòng mới được kiểm tra ngay.

> 💡 Chính phòng **reasoning** là ví dụ sống của quy trình này — nó là phòng được thêm gần nhất, và bạn có thể chỉ ra dấu vết của nó ở mọi mảnh: `ROOMS`, `THEME.reasoning`, `HINTS.reasoning`, `SEARCH_KEYWORDS.reasoning`, `BANK.reasoning`, `ASSETS` (`./rooms/reasoning.js`), và `TRACKS.dev`.
