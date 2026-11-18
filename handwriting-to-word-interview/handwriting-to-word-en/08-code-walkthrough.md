# 08 — One Conversion Walkthrough

1. `index.html` loads `src/main.js`, which initializes `HandwritingCanvas`, `HandwritingRecognizer`, `MathEngine`, `WordDocumentEditor` and `WhiteboardStudio`.
2. `canvas.js` captures pointer strokes, smooths and stores them; `getStrokesForOCR()` returns the recognition input.
3. In **text** mode, `segmentLines()` groups strokes and `recognize()` asks `/api/handwriting` for candidates; the UI lets the user choose.
4. In **math** mode, a configured Gemini key selects `recognizeMathWithAI()` on an image; otherwise `recognizeMathFormula()` analyzes strokes and assembles LaTeX. The latter may still need network access for symbol recognition.
5. `insertRecognizedText()` or `insertMathFormula()` updates the editor. KaTeX previews math; the editor saves a draft.
6. `DocxExporter.exportToDocx()` builds Word paragraphs/runs; math blocks become LaTeX text. `printToPdf()` invokes `window.print()`.

**Demo honestly:** draw a short line, correct a candidate, insert a math template, then open the exported `.docx` to show the LaTeX limitation.
