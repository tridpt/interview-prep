# 01 — Project Overview

**30-second pitch:** InkToWord Studio is a Vite/JavaScript web app for drawing with a mouse or pen, recognizing Vietnamese handwriting through Google Handwriting IME, choosing a candidate, editing the text, and exporting `.docx` or printing to PDF. Math mode combines spatial stroke analysis with optional Gemini Vision and previews LaTeX using KaTeX.

## User flow

1. Choose a pen and paper style; draw, undo/redo or erase strokes.
2. Convert (`Ctrl+Enter`), select a candidate and insert it into the editor or whiteboard.
3. Apply typography and formatting; copy rich text, export Word or print to PDF.

## Stack and launch

- Vite + ES modules; `docx`, `file-saver`, `katex`, `canvas-confetti`.
- `npm install && npm run dev`: Vite on port 5173 by default, with `/api/handwriting` proxy.
- `npm run build && npm start`: `server.js` serves `dist/` and proxies OCR, port 4173 by default.

**Be candid:** stroke OCR requires network access and depends on Google's endpoint; Vietnamese diacritics are not guaranteed. Gemini requires a separate user key. Word math exports as **plain LaTeX text**, not native Word equations; PDF uses browser printing.
