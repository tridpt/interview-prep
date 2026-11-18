# 08 — Walkthrough một lượt chuyển đổi

1. `index.html` nạp `src/main.js`; module khởi tạo `HandwritingCanvas`, `HandwritingRecognizer`, `MathEngine`, `WordDocumentEditor` và `WhiteboardStudio`.
2. `canvas.js` nhận pointer, làm mượt nét, quản lý lịch sử; `getStrokesForOCR()` trả về nét để nhận diện.
3. Chế độ **text**: `recognizer.segmentLines()` nhóm nét, `recognize()` gọi các dòng qua `/api/handwriting`; UI hiện ứng viên để chọn.
4. Chế độ **math**: nếu có Gemini key, `recognizeMathWithAI()` nhận ảnh; nếu không, `recognizeMathFormula()` phân tích cấu trúc nét rồi ghép LaTeX. Nhánh này vẫn có thể cần mạng để đọc ký tự.
5. `editor.insertRecognizedText()` hoặc `insertMathFormula()` chèn kết quả; KaTeX dựng preview, editor lưu nháp.
6. `DocxExporter.exportToDocx()` chuyển nội dung thành các đoạn/runs Word; `.word-math-block` thành chuỗi LaTeX. `printToPdf()` gọi `window.print()`.

**Khi demo:** viết một dòng ngắn, sửa gợi ý, chèn một mẫu toán, xuất `.docx`; mở file để chỉ ra giới hạn LaTeX thay vì nói quá khả năng export.
