# 08 — Cheat Sheet (last-minute skim)

## 30-second pitch
> Art Shield protects artists' paintings from being scraped for AI training. It adds a tiny, human-invisible perturbation that makes CLIP extract wrong features. I hand-rolled PGD in PyTorch, used EoT so the noise survives compression/resize, and perceptual masking to hide it. It has tools to measure robustness and transfer — and I'm upfront it's not absolute.

## Core formula
- **Objective**: minimize `cosine(encode(image+δ), encode(image))`, with `‖δ‖∞ ≤ ε`.
- **PGD update**: `δ -= step_size · sign(∇δ similarity)` → project to `[−ε,ε]` → keep image ∈ `[0,1]`.
- **Protection threshold**: cosine to original **< 0.5** (lower is better; negative = very strong).

## Numbers to remember
| | Robustness | SSIM |
|--|--|--|
| Naive PGD ε=0.03 | 0/9 ❌ | 0.83 |
| EoT ε=0.06 | 9/9 ✅ | 0.60 |
| EoT+perceptual ε=0.04 | 9/9 ✅ | 0.79 |
| Real image EoT+perceptual ε=0.08 | 9/9 ✅ | **0.94** (cosine −0.42) |

Transfer: targeting B/32 → on B/32 cosine **−0.49** (good), on B/16 **+0.84** (useless).

## One-line glossary
- **Embedding**: 512-D vector describing an image (CLIP).
- **Cosine**: direction similarity; 1=same, <0.5=protected.
- **PGD**: iterative gradient + projection to craft noise.
- **ε**: max noise budget (L-inf).
- **EoT**: optimize noise over many transforms → robust.
- **STE**: gradients through rounding (`x+(q-x).detach()`).
- **Perceptual mask**: hide noise in detailed regions (local std).
- **Transferability**: noise carrying to other models (here: poor).

## File map
| File | Role |
|------|------|
| `clip_model.py` | Wrap CLIP, encode keeping pixel gradients |
| `attack.py` | `pgd_protect`, `pgd_protect_eot`, `..._multi` |
| `diff_transforms.py` | DIFFERENTIABLE transforms (train EoT) |
| `transforms.py` | PIL transforms (TEST robustness, 9 of them) |
| `perceptual.py` | Perceptual masking map |
| `metrics.py` | PSNR, SSIM |
| `pipeline.py` | `protect_image`, `run_robustness`, lazy CLIP singleton |
| `backend/main.py` | FastAPI `/api/protect`, `/api/verify` |
| `backend/config.py` | PRESETS Light/Medium/Strong, MAX_IMAGE_SIDE=512 |

## Protect flow (web)
`POST /api/protect` → `_load_and_fit` (≤512px) → `protect_image` → `perceptual_mask` → `pgd_protect_eot` → measure cosine/PSNR/SSIM → save original+protected → JSON.

## Robustness flow
`POST /api/verify` (uid) → read 2 images → `run_robustness` → apply 9 wash-off transforms to the protected image → count cosine<0.5 → held/total table.

## Presets
| | ε | steps | eot_samples |
|--|--|--|--|
| Light | 0.04 | 60 | 4 |
| Medium | 0.06 | 90 | 4 |
| Strong | 0.09 | 140 | 5 |

## Go-to line on limits
> "No tool is 100% immune, including Glaze. Noise targeting one model doesn't transfer well to a different architecture — I have an experiment proving it. I treat it as a friction layer that raises the cost for abusers, combined with watermarks and terms of use."

## Common pitfalls
- StaticFiles `/` mount must be **last** (after /api).
- `F.interpolate` (differentiable) ≠ PIL resize (cuts gradients).
- `round` blocks gradients → need STE.
- Measure with **cosine**, not classification label flips.
- EoT is slow because it encodes many times per step → 512px limit.
