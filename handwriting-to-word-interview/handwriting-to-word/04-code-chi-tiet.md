# 04 — Code chi tiết

| File | Vai trò, hàm đáng đọc |
|---|---|
| `src/canvas.js` | `HandwritingCanvas`: Pointer Events, `drawSmoothSegment`, `getStrokesForOCR`, `undo`, `redo`. |
| `src/recognizer.js` | `HandwritingRecognizer.segmentLines`, `recognize`, `recognizeSingleLine`, `recognizeImageWithGemini`. |
| `src/math-engine.js` | `MathEngine.detectExponentStructure`, `detectFractionStructure`, `recognizeMathFormula`, `recognizeMathWithAI`; `MATH_TEMPLATES`. |
| `src/editor.js` | `WordDocumentEditor`: `insertRecognizedText`, `insertMathFormula`, `saveDraft`, `updateStats`. |
| `src/whiteboard.js` | `WhiteboardStudio`: thẻ văn bản/toán, kéo thả, lưu/khôi phục state, xuất ảnh. |
| `src/docx-export.js` | `DocxExporter`: tạo `Document`/`Paragraph`/`TextRun`, sao chép rich text, `printToPdf`. |
| `src/main.js` | Kết nối UI, chọn chế độ text/math, gợi ý, xử lý lỗi và xuất file. |
| `vite.config.js`, `server.js` | Proxy Google Handwriting IME cho dev/production; server production phục vụ build tĩnh. |

Lưu ý ở `DocxExporter`: nhánh `.word-math-block` lấy `data-latex` và ghi `TextRun` văn bản. Đừng nói export phương trình Word/OMML.
