# 01 — Project Overview

## 1. What is this project? (explained for a complete beginner)

Imagine you are an **artist**. You post your paintings online. Today, AI companies **scrape** millions of artworks from the Internet to **train** automatic image-generation models. The result: AI can imitate your style without consent or payment.

**Art Shield** helps artists fight back. The idea: before posting, you add a **tiny perturbation** (an *adversarial perturbation*) to the image. This perturbation:

- **Is nearly invisible to humans**: the image looks identical to the original.
- **Changes the targeted CLIP model's features** in this experiment; preventing image-generation models from learning or copying a style has not been tested.

This is the same idea behind well-known tools like **Glaze** and **Nightshade** from the University of Chicago. Art Shield is a **research demo** reproducing the technique.

## 2. What problem does it solve?

| Problem | How Art Shield handles it |
|---------|---------------------------|
| Art scraped to train AI without permission | Experiment with noise targeting a specific image encoder; no guarantee across other models |
| Noise must be invisible to humans | Measure with PSNR/SSIM + hide noise in detailed regions (perceptual masking) |
| Scrapers can compress/resize to "wash off" noise | Train the noise with EoT so it survives transforms |
| Non-technical users | Drag-drop web app with Light/Medium/Strong presets |

## 3. Project stages

The project is built in three clear stages (per the README):

- **Stage 1 — Prove the technique**: noise makes CLIP misread, while the image is nearly unchanged (high PSNR/SSIM). File `protect.py`.
- **Stage 2 — Measure & upgrade robustness**: check whether the noise survives compression/resize/blur (`robustness.py`); upgrade from naive PGD (0/9 survival) to **EoT** (9/9) plus **perceptual masking**.
- **Stage 3 — Web app**: a Vietnamese UI for artists (`backend/` + `frontend/`).

## 4. Main technologies

| Component | Tech | Role |
|-----------|------|------|
| Target AI model | **CLIP** (OpenAI, via `transformers`) | The "AI eye" we fool |
| Deep learning / optimization | **PyTorch** | Compute gradients to craft noise |
| Noise algorithm | **PGD** + **EoT** (hand-rolled) | Optimize the adversarial perturbation |
| Image quality | **PSNR, SSIM** (`scikit-image`) | Prove the noise is hard to see |
| Web backend | **FastAPI** + `uvicorn` | `/api/protect`, `/api/verify` |
| Web frontend | Vanilla HTML/CSS/JS | Drag-drop, show results |
| Image processing | **Pillow (PIL)**, NumPy | Read/write, transform images |

## 5. Directory structure (trimmed)

```
art-shield/
├── protect.py          # CLI: add protective noise + report
├── robustness.py       # CLI: test robustness vs compress/resize/blur
├── eval_transfer.py    # Experiment: does noise "transfer" to other models?
├── make_sample.py      # Create a sample image to test
├── shield/             # SHARED CORE
│   ├── clip_model.py       # Wrap CLIP, encode image/text, classify
│   ├── attack.py           # PGD + PGD-EoT + ensemble (craft noise)
│   ├── diff_transforms.py  # DIFFERENTIABLE transforms for EoT
│   ├── transforms.py       # PIL transforms to TEST robustness
│   ├── perceptual.py        # Perceptual masking map
│   ├── pipeline.py          # Shared core for CLI + web
│   └── metrics.py           # PSNR / SSIM
├── backend/            # Web app (FastAPI)
│   ├── main.py
│   └── config.py            # Light/Medium/Strong presets, paths
└── frontend/           # Web UI (index.html / app.js / style.css)
```

## 6. What makes it notable

- **Hand-rolled PGD and EoT** in PyTorch (no off-the-shelf adversarial library) → shows deep understanding of the gradient mechanics.
- **Honest about limits**: `eval_transfer.py` compares CLIP-B/32 and CLIP-B/16. It **does not run GPT-4o** or test whether an image-generation model can learn a style.
- **Correct metric**: uses **cosine similarity to the original** as the main measure (threshold < 0.5), not "did the classification label flip" (which is misleading).

## 7. 30-second pitch

> "Art Shield protects artists' paintings from being scraped for AI training. I add a tiny perturbation, invisible to humans, that makes CLIP extract very different features. I hand-rolled PGD in PyTorch to optimize the noise, used EoT so it survives compression/resize, and perceptual masking to hide it. I also built tools to measure robustness and transfer — and I'm upfront that the protection is not absolute, just like Glaze/Nightshade."
