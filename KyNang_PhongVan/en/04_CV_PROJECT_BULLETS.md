# 4. Writing the Projects section of your CV

The projects section is what recruiters read most for a fresher/junior (you don't have much work experience yet). Writing it well = getting the interview.

## Structure per project
```
[Project name] — [one line: what it is] · [main stack]
- Bullet 1: what you built + tech + (a number if possible)
- Bullet 2: one hard/standout technical point
- Bullet 3: quality/process (tests, CI, security...)
[GitHub / demo link]
```

## Bullet-writing rules
- **Start with a strong verb**: Built / Designed / Implemented / Optimized / Integrated / Tested.
- **Have "what" + "with what" + "result"**: "Implemented the TOTP algorithm per RFC 6238 with Python's standard library, verified by official test vectors."
- **Use numbers**: lines of code, number of tests, features, performance.
- **Avoid vague**: replace "did the frontend" → "built a drag-and-drop UI in vanilla JS, no framework".

## Sample bullets (from the projects in this pack)

**OTP Vault — A local 2FA (TOTP) manager · Python, FastAPI, cryptography**
- Implemented the **TOTP algorithm per RFC 6238** (HMAC + dynamic truncation) using only the standard library; verified with the **official test vectors** for SHA1/256/512.
- Designed a **"no stored key" encryption model**: the key is derived from a master password via **scrypt**, the vault is encrypted with **Fernet (AES)**, and the key only lives in RAM.
- Wrote a test suite with **property-based testing (hypothesis)** and API tests via FastAPI's TestClient.
- GitHub: …

**Taste Map — A food-map web app organized by personal taste · Vanilla JS, Leaflet, IndexedDB**
- Built an interactive map (Leaflet + markercluster + heatmap), with a "where to go today" mood suggester using a weighted scoring algorithm.
- **Two-tier** storage (localStorage for text, IndexedDB for images) to beat the 5MB limit; **AES-GCM encrypted** backups.
- **47 Playwright tests** (E2E, mocked fetch) + **GitHub Actions CI/CD** that deploys only when tests pass.

**Word Checker — A Word formatting/spelling checker · Python, python-docx, Flask**
- Resolved the **"effective font"** across 4 OOXML inheritance tiers for accurate font checks.
- A layered architecture (one module per check, errors as a shared `Issue`) reused by both a CLI and a web app; packaged into an `.exe` (PyInstaller).
- Focused on **false-positive reduction** (skipping math formulas...) with dedicated "must-not-flag" tests.

**Art Shield — Protecting art from AI via adversarial noise · Python, PyTorch, CLIP, FastAPI**
- Implemented **PGD** and **EoT (Expectation over Transformation)** from scratch in PyTorch to craft adversarial noise that corrupts an image's CLIP features; raised robustness against compression/resize from **0/9 to 9/9**.
- Added **perceptual masking** to hide noise in detailed regions (SSIM 0.94 — near-invisible) and an **STE** trick to flow gradients through quantization.
- Measured **transferability** across models (B/32 vs B/16) and stayed honest about limits; a drag-and-drop FastAPI web app with Light/Medium/Strong presets.

**AI Usage Monitor — Measuring AI token/credit usage from local logs · Python, FastAPI, tiktoken**
- Built a **defensive log scanner** (regex filter, 80MB cap, fingerprint dedup) detecting usage objects **recursively** across schemas (Claude/Kiro/Antigravity).
- Counts tokens accurately with **tiktoken** using a model→encoding map + `lru_cache`; clearly distinguishes tokens vs credits and **doesn't fabricate** numbers when a vendor exposes only credits.
- Fully **local, read-only**, with XSS protection on the frontend.

**Voice Clone Web — Vietnamese voice cloning (TTS) · Python, FastAPI, viXTTS, Web Audio API**
- Integrated **viXTTS** for zero-shot voice cloning from a few seconds of audio (conditioning latents); a **pluggable** + **lazy-load** + soft-import engine.
- Hand-rolled a **browser-side WAV encoder** (Web Audio → downsample → 44-byte header → int16) so the backend just saves files.
- A **dataset → fine-tune** pipeline (200 phoneme-covering sentences, LJSpeech export, Colab notebook); added a **mandatory consent step** for ethics.

## Tips
- Order projects with the **strongest / most JD-relevant first**.
- 2–4 bullets per project is enough; quality over quantity.
- Always include a **GitHub link** (and demo if any) — recruiters will click.
- Make sure you can **explain every bullet** when probed — don't write what you don't understand.

> 💡 A good bullet is "question bait": write what you WANT to be asked, to show depth (e.g. "effective font", "no stored key", "rolling summary").
