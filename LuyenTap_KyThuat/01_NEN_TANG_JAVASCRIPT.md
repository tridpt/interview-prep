# 1. Nền tảng JavaScript

Ôn các điểm hay bị hỏi, gắn với cách dùng thật trong các dự án vanilla JS (cap-cuu-101, quan-quen-map, agent-talk frontend, word-checker web).

## 1. var / let / const & scope
- `var`: phạm vi hàm (function scope), bị "hoisting". **Tránh dùng.**
- `let`/`const`: phạm vi khối (block scope). `const` không gán lại (nhưng object vẫn sửa được nội dung).
- **Closure**: hàm "nhớ" biến ở scope bao ngoài. (Ví dụ: trong app.js, các handler sự kiện truy cập biến state nhờ closure.)

## 2. == vs ===
- `==` so sánh có ép kiểu (`0 == "0"` → true) → dễ bug.
- `===` so sánh nghiêm ngặt (kiểu + giá trị). **Luôn ưu tiên `===`.**

## 3. Bất đồng bộ: callback → Promise → async/await
- **Event loop**: JS đơn luồng; tác vụ I/O chạy nền, kết quả vào hàng đợi, xử lý khi stack rỗng.
- **Promise**: đại diện giá trị tương lai (`pending → fulfilled/rejected`).
- **async/await**: cú pháp gọn cho Promise. (Ví dụ: `const res = await fetch(...)` trong mọi frontend của bộ.)
- **Đọc streaming**: `response.body.getReader()` + vòng `while` đọc từng chunk (agent-talk hiển thị từng chữ LLM gõ ra).

## 4. DOM & sự kiện
- `document.getElementById/querySelector`, tạo phần tử, `addEventListener`.
- **Event delegation**: gắn 1 listener lên container, dùng `event.target.closest("[data-...]")` thay vì gắn cho từng phần tử (quan-quen-map dùng kỹ thuật này nhiều).
- **`<template>`**: nhân bản phần tử lặp lại bằng `cloneNode(true)` (agent-talk tạo thẻ agent).

## 5. this, arrow function
- Arrow function **không có `this` riêng** — lấy `this` từ scope bao ngoài. Hữu ích trong callback.
- Hàm thường: `this` phụ thuộc cách gọi.

## 6. Mảng & object hay dùng
- `map`/`filter`/`reduce`/`find`/`sort`/`some`/`every`.
- Spread `...`, destructuring `{a, ...rest}` (vd loại trường `memory` khi chia sẻ: `({memory, ...rest}) => rest`).
- `JSON.parse`/`JSON.stringify`.

## 7. Web API trình duyệt (rất hay hỏi vì các dự án dùng nhiều)
- **fetch + Streams**, **localStorage** (text, ~5MB), **IndexedDB** (lớn, async — quan-quen-map lưu ảnh).
- **Service Worker** (PWA, cache offline), **Canvas**, **Web Audio** (cap-cuu-101 tự synth âm thanh), **Web Speech** (TTS), **getUserMedia** (camera quét QR — totp-manager).

## Câu hỏi mẫu
- "Giải thích event loop?" → đơn luồng + hàng đợi callback/microtask.
- "Closure là gì, dùng khi nào?" → hàm nhớ biến ngoài; vd đếm, ẩn dữ liệu.
- "localStorage vs IndexedDB?" → text/đồng bộ/5MB vs object/async/lớn.
- "Vì sao dùng `===`?" → tránh ép kiểu ngầm gây bug.
- "Làm sao đọc dữ liệu streaming từ server?" → getReader() + decode từng chunk.

> 💡 Luôn **nối với dự án**: "Em dùng IndexedDB trong quan-quen-map vì ảnh base64 làm đầy localStorage 5MB."
