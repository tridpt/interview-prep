# 06 — Interview Questions

**1. Does OCR work offline?** No. Stroke recognition calls Google Handwriting IME; spatial math mode still needs the recognizer for characters. Editing and drafts are local, but OCR is not offline.

**2. Why store strokes instead of just images?** Coordinates enable line grouping, undo, redraw and vector-based API requests. Raster images are used for Vision and image export.

**3. What does the proxy do?** It provides a stable `/api/handwriting` path in Vite and the Node server and reduces direct-browser CORS dependence; it does not replace Google's service.

**4. How does 2D math work?** Spatial heuristics classify fractions/exponents and other patterns, recognize symbols and assemble LaTeX. Quick templates and optional Gemini Vision help; complex formulas can be wrong.

**5. Is Word/PDF output identical to the editor?** Not always. `docx` creates Word paragraphs, with math as LaTeX text; PDF invokes browser printing. Native editable equations would require OMML conversion.

**6. Privacy risks?** Strokes/images go to external OCR; optional Gemini uses the user's key and sends images externally. Do not use confidential source material without reviewing provider policies.

**7. Tests and CI?** The package offers a build, but this repository currently has no automated test/CI suite for this flow. Priorities: segmentation, 2D math, export and network failures.
