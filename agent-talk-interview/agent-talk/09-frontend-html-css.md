# 9. Phụ lục: Frontend HTML & CSS (đọc trực tiếp file thật)

> ⚠️ File này được viết sau khi **đọc trực tiếp `frontend/index.html` và `frontend/style.css`** — hai file chưa được phân tích ở các phần trước. Đây là phần bổ sung để bộ tài liệu agent-talk **đầy đủ 100% mọi file**.

---

## 9.1. `index.html` — Cấu trúc giao diện

`index.html` rất gọn (~110 dòng), gồm 3 phần: header, main, và một `<template>`.

### Bố cục tổng thể
```
<header>  → tiêu đề + mô tả
<main>
  <section class="setup">     → khu cấu hình (agent, chủ đề, tùy chọn, nút)
  <section class="chat">      → khung hiển thị hội thoại (rỗng, app.js đổ vào)
  <div class="interject">     → ô "đạo diễn" xen vào
<template id="agentTpl">      → khuôn một thẻ agent (nhân bản khi thêm agent)
<script src="/personas.js">
<script src="/app.js">
```

### Khu `setup` — các id mà `app.js` thao tác
Đây là bảng tra cứu mọi phần tử có `id` (vì `app.js` dùng `$ = id => document.getElementById(id)`):

| id / class | Loại | Vai trò |
|-----------|------|---------|
| `#agents` | div | Nơi chứa các thẻ agent (app.js `appendChild` vào đây) |
| `#addAgentBtn` | button | Thêm agent |
| `#topic` | input | Chủ đề (mặc định "Liệu AI có thể có cảm xúc thật sự không?") |
| `#orderMode` | select | `round` (luân phiên) / `auto` (AI tự chọn) / `debate` (tranh biện) |
| `#maxTurns` | input number | Số lượt mỗi lần (mặc định 12) |
| `#delay` | input number | Tốc độ giây/lượt (mặc định 1) |
| `#temp` | input number | Độ sáng tạo temperature (mặc định 0.9) |
| `#ttsOn` `#memOn` `#searchOn` | checkbox | Bật đọc to / tự cập nhật bộ nhớ / cho phép tra cứu web |
| `#startBtn` `#continueBtn` `#stopBtn` `#clearBtn` | button | Điều khiển vòng lặp |
| `.save[data-fmt]` | button | Lưu file .txt/.md/.json (đọc `data-fmt`) |
| `#copyBtn` `#shareBtn` `#scoreBtn` | button | Sao chép / chia sẻ link / chấm điểm |
| `#sessionInfo` | span | Hiện trạng thái phiên đã lưu |
| `#forgetBtn` | button | Xoá phiên localStorage |
| `#scoreResult` `#leaderboard` | div | Nơi vẽ kết quả chấm điểm & bảng xếp hạng (mặc định `hidden`) |
| `#chat` | section | Khung chat |
| `#interjectInput` `#interjectBtn` | input/button | Đạo diễn xen vào |

➡️ **Liên hệ với file 05 (frontend):** mọi `id` ở đây chính là thứ `app.js` tham chiếu. Khi `refreshUi()` bật/tắt `disabled`, nó duyệt đúng các id này.

### `<template id="agentTpl">` — khuôn thẻ agent
`app.js` `addAgentCard()` nhân bản template này. Bên trong có các class **app.js đọc/ghi**:
- `.a-name` (tên), `.a-provider` (select bộ não LLM), `.a-tpl` (select tính cách mẫu), `.a-persona` (textarea tính cách), `.a-goal` (mục tiêu ngầm).
- `.mem-box` là một `<details>` (gập/mở) chứa `.a-memory` (textarea bộ nhớ) + `.mem-clear` (nút xoá bộ nhớ).
- `.dot` là chấm màu (đổi theo `--c` để phân biệt agent).

> 💡 **Vì sao dùng `<template>`?** Thẻ `<template>` không render cho đến khi được nhân bản bằng JS (`content.firstElementChild.cloneNode(true)`). Đây là cách tạo phần tử lặp lại **không cần framework**, sạch hơn là viết HTML bằng chuỗi.

### Điểm cần lưu ý khi phỏng vấn
- Script nạp theo thứ tự: `personas.js` (định nghĩa `PERSONA_TEMPLATES`) **trước** `app.js` (dùng nó).
- File HTML **không chứa logic** — toàn bộ hành vi ở `app.js`. HTML chỉ là khung khai báo.

---

## 9.2. `style.css` — Giao diện

`style.css` (~270 dòng) là CSS thuần, dùng **CSS custom properties (biến)** và bố cục flex/grid. Không framework CSS.

### Biến theme & nền tảng
```css
:root { --bg:#0f1220; --panel:#1a1f35; --text:#e8eaf2; --muted:#9aa3c0; }
* { box-sizing: border-box; margin:0; padding:0; }
body { max-width: 920px; margin: 0 auto; ... }   /* canh giữa, giới hạn bề ngang */
```
Theme **tối** (dark). Biến CSS giúp đổi màu tập trung một chỗ.

### Các khối chính
- **`.agents`**: `display: grid` với `repeat(auto-fill, minmax(240px, 1fr))` → các thẻ agent **tự xếp cột** theo bề ngang màn hình (responsive không cần media query cho phần này).
- **`.agent-card`**: có biến cục bộ `--c: #3b82f6` (màu agent); `border-top` và `.dot` dùng `var(--c)`. `app.js` đặt `node.style.setProperty("--c", COLORS[i])` để mỗi agent một màu.
- **`.bubble`**: bong bóng chat, có `animation: pop .25s` khi xuất hiện.
  - `.bubble.agent`: căn trái, viền trái màu agent.
  - `.bubble.director`: căn giữa, nền nâu, in nghiêng (lời đạo diễn).
  - `.bubble.error`: nền đỏ (thông báo lỗi).
  - `.bubble.phase`: banner giai đoạn tranh biện (chữ hoa, mờ).

### Hiệu ứng "đang gõ" (streaming) — chi tiết hay
```css
.bubble.speaking .txt::after {
  content: "▋"; animation: blink 1s steps(2, start) infinite;
}
@keyframes blink { to { visibility: hidden; } }
```
➡️ Khi bong bóng có class `speaking` (lúc đang stream), một con trỏ nhấp nháy `▋` hiện ở cuối — **mô phỏng cảm giác đang gõ chữ**. `app.js` thêm/gỡ class `speaking` quanh lúc gọi `streamTurn`.

### Khối chấm điểm & bảng xếp hạng
- `.score-bar span` / `.crit-bar i`: thanh điểm, `width` được `app.js` đặt theo % (transition mượt `.4s`).
- `.score-item.winner`: viền vàng (`box-shadow ... inset`) cho người thắng.
- `.crit-grid`: lưới 2 cột 4 tiêu chí (logic/bằng chứng/sáng tạo/thái độ); xuống 1 cột ở mobile.
- `.lb-table`: bảng xếp hạng, hàng đầu (top 1) chữ to hơn.

### Responsive
```css
@media (max-width: 640px) {
  .agents { grid-template-columns: 1fr; }  /* 1 cột trên mobile */
  .bubble { max-width: 92%; }
  .crit-grid { grid-template-columns: 1fr; }
}
```

### Footnote nguồn tra cứu
`.bubble .sources`: viền trên nét đứt, chữ nhỏ; link `a` màu xanh nhạt `#7dd3fc`. `app.js` `renderSources()` đổ link vào đây.

---

## 9.3. Tổng kết: liên kết HTML/CSS với JS

| Việc | HTML/CSS lo | app.js lo |
|------|-------------|-----------|
| Khung & bố cục | `index.html` + grid/flex | — |
| Màu mỗi agent | biến `--c` trong CSS | đặt `--c` theo chỉ số |
| Bong bóng chat | class `.bubble.*` + animation | tạo phần tử, thêm class, đổ text |
| Hiệu ứng đang gõ | `.speaking` + `@keyframes blink` | thêm/gỡ class `speaking` khi stream |
| Thẻ agent lặp lại | `<template id="agentTpl">` | `cloneNode` + gắn sự kiện |
| Thanh điểm | width + transition trong CSS | đặt `style.width = pct%` |
| Ẩn/hiện kết quả | thuộc tính `hidden` | gỡ/đặt `hidden` |

> 💡 **Câu hỏi phỏng vấn:** "Bạn làm responsive và đa màu mà không framework thế nào?" → CSS Grid `auto-fill` cho layout tự co, CSS custom properties (`--c`) cho màu động đặt từ JS, và `@media` cho mobile. Tất cả thuần CSS, nhẹ và không build.
