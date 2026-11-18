# 03 — System Architecture

```text
index.html → src/main.js
  ├─ canvas.js: pointer input → strokes and undo/redo history
  ├─ recognizer.js: line segmentation → handwriting candidates
  ├─ math-engine.js: 2D structure / optional Gemini → LaTeX
  ├─ editor.js: typography, KaTeX, localStorage draft
  ├─ whiteboard.js: cards and whiteboard
  └─ docx-export.js: Word, Clipboard, browser print/PDF

Development: browser → Vite /api/handwriting → Google Handwriting IME
Production: browser → server.js /api/handwriting → Google Handwriting IME
```

`vite.config.js` sets the dev proxy. `server.js` only serves built assets and proxies OCR. If the proxy fails, the recognizer tries a direct endpoint (which may be blocked by CORS/network policy). Optional Gemini Vision is called from the browser with the user's key.

The application is predominantly client-side, with no server account database. Do not describe `server.js` as a self-trained OCR model.
