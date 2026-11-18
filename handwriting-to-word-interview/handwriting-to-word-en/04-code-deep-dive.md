# 04 — Code Deep Dive

| File | Responsibility / functions to read |
|---|---|
| `src/canvas.js` | `HandwritingCanvas`: Pointer Events, `drawSmoothSegment`, `getStrokesForOCR`, `undo`, `redo`. |
| `src/recognizer.js` | `HandwritingRecognizer.segmentLines`, `recognize`, `recognizeSingleLine`, `recognizeImageWithGemini`. |
| `src/math-engine.js` | `MathEngine.detectExponentStructure`, `detectFractionStructure`, `recognizeMathFormula`, `recognizeMathWithAI`; `MATH_TEMPLATES`. |
| `src/editor.js` | `WordDocumentEditor`: `insertRecognizedText`, `insertMathFormula`, `saveDraft`, `updateStats`. |
| `src/whiteboard.js` | `WhiteboardStudio`: text/math cards, dragging, saved state and image export. |
| `src/docx-export.js` | `DocxExporter`: `Document`/`Paragraph`/`TextRun`, rich clipboard and `printToPdf`. |
| `src/main.js` | UI wiring, text/math mode, candidates, errors and exports. |
| `vite.config.js`, `server.js` | Dev/production proxy to Google Handwriting IME; built-file server. |

In `DocxExporter`, `.word-math-block` reads `data-latex` and writes a text `TextRun`: it does **not** create Word OMML equations.
