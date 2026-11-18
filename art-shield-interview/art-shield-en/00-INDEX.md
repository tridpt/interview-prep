# 📑 Index — Art Shield Interview Prep

Welcome! This set helps you **understand Art Shield deeply** for an interview, even if you know nothing about AI or adversarial noise. Reading `00 → 08` in order is best.

> Art Shield is a tool that **protects artists' paintings from being scraped to train AI**, by adding a tiny "adversarial perturbation" that humans barely notice but that makes an AI model "see" the painting wrong.

## Reading order

| File | Title | Content | When |
|------|-------|---------|------|
| `00` | **Index** (this file) | Map of all docs | First |
| `01` | [Project overview](01-project-overview.md) | What it is, the problem, main tech | Big picture |
| `02` | [Foundational concepts](02-foundational-concepts.md) | CLIP, embeddings, adversarial noise, PGD, EoT, PSNR/SSIM — from zero | Must-read theory |
| `03` | [System architecture](03-system-architecture.md) | Layers, data flow, how modules fit | After concepts |
| `04` | [Core internals](04-core-internals.md) | Function-by-function: CLIP wrapper, PGD/EoT attack, perceptual mask, transforms, metrics | The technical guts |
| `05` | [Web app detail](05-web-app-detail.md) | FastAPI backend + drag-drop frontend | The user-facing demo |
| `06` | [Standout features](06-standout-features.md) | Point-scorers: EoT, perceptual masking, transfer eval | Pick your highlights |
| `07` | [Interview questions](07-interview-questions.md) | 60+ Q&A, basic to hard | Practice answering |
| `08` | [Cheat sheet](08-cheat-sheet.md) | One-page recap for last-minute | Right before the interview |

## After reading you should be able to

1. **Pitch the project in 2 minutes**: what, for whom, with what tech.
2. **Explain the principle**: why a tiny perturbation fools the AI while humans see no change.
3. **Handle follow-ups**: "why EoT?", "does the noise transfer to other models?", "what are the limits?".
4. **Be honest about limits**: this is research demo, not "100% immune" — saying this well scores points.

## One line to remember

> "Art Shield optimizes a tiny perturbation on an image so that the image's **embedding in CLIP's space is pushed far away from the original** — the AI sees something else entirely, while humans don't. To survive compression/resize I use EoT; to hide the noise I use a perceptual mask."
