# 07 — Cheat Sheet

- **One line:** handwriting → Google IME → candidates → editor/whiteboard → `.docx` or print-to-PDF.
- **Stack:** Vite, vanilla ES modules, `docx`, `file-saver`, KaTeX; Node built-file/OCR proxy.
- **Entry:** `src/main.js`; strokes in `canvas.js`, OCR in `recognizer.js`, math in `math-engine.js`.
- **Ports:** dev 5173, production 4173 by default; `/api/handwriting` proxies to Google.
- **Math:** 2D spatial recognition or optional Gemini; Word exports plain LaTeX text.
- **State:** editor draft in `localStorage`; Gemini key in `sessionStorage`.
- **Run:** `npm install`, `npm run dev`; production `npm run build`, `npm start`.
- **Limits:** no offline OCR, no guaranteed perfect diacritics, no automated test suite in repo.
