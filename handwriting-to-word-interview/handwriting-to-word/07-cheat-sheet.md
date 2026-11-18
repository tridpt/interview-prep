# 07 — Cheat sheet

- **Một câu:** viết tay → Google Handwriting IME → gợi ý → editor/whiteboard → `.docx` hoặc print-to-PDF.
- **Stack:** Vite + vanilla ES modules + `docx` + `file-saver` + KaTeX; Node `server.js` phục vụ build/proxy.
- **Entry:** `src/main.js`; canvas ở `src/canvas.js`, OCR ở `src/recognizer.js`, toán ở `src/math-engine.js`.
- **URL:** dev 5173, production 4173 mặc định; `/api/handwriting` chuyển tiếp sang Google.
- **Math:** nhận diện cấu trúc 2D + ứng viên; Gemini Vision tùy chọn; Word xuất LaTeX thuần.
- **State:** bản nháp editor `localStorage`; Gemini key `sessionStorage`.
- **Chạy:** `npm install`, `npm run dev`; production `npm run build`, `npm start`.
- **Giới hạn:** không OCR offline, không bảo đảm dấu đúng 100%, chưa có test tự động trong repo.
