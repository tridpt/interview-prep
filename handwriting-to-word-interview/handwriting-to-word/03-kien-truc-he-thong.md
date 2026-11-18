# 03 — Kiến trúc hệ thống

```text
index.html → src/main.js
  ├─ canvas.js: pointer → nét + lịch sử undo/redo
  ├─ recognizer.js: phân dòng → ứng viên từ handwriting API
  ├─ math-engine.js: cấu trúc 2D / Gemini tùy chọn → LaTeX
  ├─ editor.js: nội dung, typography, KaTeX, bản nháp localStorage
  ├─ whiteboard.js: card và bảng trắng
  └─ docx-export.js: Word, Clipboard, browser print/PDF

Dev: browser → Vite /api/handwriting → Google Handwriting IME
Prod: browser → server.js /api/handwriting → Google Handwriting IME
```

`vite.config.js` định nghĩa proxy dev; `server.js` chỉ phục vụ `dist/` và proxy OCR sau build. Khi proxy lỗi, recognizer thử endpoint trực tiếp (có thể bị CORS/chính sách mạng chặn). Gemini Vision được gọi từ client với key người dùng nhập, không đi qua `server.js`.

**Thiết kế:** tách canvas, nhận diện, toán, biên tập, xuất file để mỗi luồng có trách nhiệm riêng. Đây là app hướng client, không có tài khoản/DB server. Đừng mô tả `server.js` là một hệ thống OCR tự huấn luyện.
