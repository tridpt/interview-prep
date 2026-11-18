# 02 — Foundational Concepts

## Vector strokes versus raster images

`HandwritingCanvas` stores positions and timing for strokes and redraws them on a bitmap canvas. Strokes support undo/redo and the handwriting API; `toDataURL()` produces an image for Vision or image export. Do not conflate stroke recognition with OCR of an uploaded photo.

## Line segmentation and candidates

`HandwritingRecognizer.segmentLines()` groups strokes using vertical distance and overlap, taking accent marks into account. `recognize()` requests candidates for each line. This heuristic can merge nearby lines; the candidate bar lets users correct the result.

## Math

`MathEngine` detects spatial relationships such as fractions and exponents and combines them with character recognition. LaTeX encodes the formula; KaTeX **renders a preview**, not a native Word equation. Gemini Vision is an optional alternative.

## Privacy and reliability

Recognition is not offline: strokes/images go to Google; optional Gemini sends images to its provider. The Gemini key is held in browser `sessionStorage`, not sent via this app's OCR proxy. Avoid sensitive material and review provider terms.
