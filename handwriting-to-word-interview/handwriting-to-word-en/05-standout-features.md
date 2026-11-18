# 05 — Standout Features and Trade-offs

1. **Stroke history:** storing strokes instead of only an image enables undo/redo, redraw and recognition; erasing and redraw logic become more complex.
2. **Multiline segmentation:** `segmentLines()` keeps separate lines apart, but geometry alone is imperfect, so user correction remains important.
3. **Two math routes:** spatial heuristics need no Gemini key but still use character recognition via the handwriting API; optional Vision handles harder shapes at the cost of external data transfer.
4. **Editing and export:** typography and `.docx` generation run in the browser; PDF is print-to-PDF, while Word equations are plain LaTeX text.
5. **Shared proxy path:** `/api/handwriting` works in dev and production, but still depends on the external Google IME endpoint.
