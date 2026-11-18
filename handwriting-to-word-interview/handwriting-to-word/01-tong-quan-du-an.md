# 01 — Tổng quan dự án

**Pitch 30 giây:** InkToWord Studio là web app Vite/JavaScript để viết bằng chuột hoặc bút trên canvas, nhận diện nét chữ tiếng Việt qua Google Handwriting IME, chọn gợi ý rồi biên tập và xuất văn bản `.docx` hoặc in PDF. Chế độ toán nhận dạng cấu trúc nét 2D hoặc dùng Gemini Vision tùy chọn, hiển thị bằng KaTeX.

## Luồng người dùng

1. Chọn bút/nền giấy, viết trên canvas; có undo/redo và xóa nét.
2. Chuyển đổi (`Ctrl+Enter`) → chọn ứng viên nhận diện → chèn vào trình soạn thảo hoặc bảng trắng.
3. Chọn typography, định dạng nội dung, sao chép rich text, xuất Word hoặc in ra PDF.

## Stack và cách chạy

- Vite + ES modules; `docx`, `file-saver`, `katex`, `canvas-confetti`.
- `npm install && npm run dev`: Vite mặc định cổng 5173, proxy `/api/handwriting`.
- `npm run build && npm start`: `server.js` phục vụ `dist/` và proxy OCR, cổng 4173 mặc định.

**Giới hạn cần nói thẳng:** OCR nét chữ cần mạng và phụ thuộc endpoint Google; không hứa đọc đúng mọi dấu tiếng Việt. Gemini dùng key riêng khi người dùng bật. Công thức xuất `.docx` là chuỗi LaTeX, **chưa** là equation Word gốc; PDF dùng `window.print()`.
