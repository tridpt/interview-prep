# 03 — System Architecture

## 1. Big picture

Art Shield has **two usage faces** sharing one core:

```
                    ┌─────────────────────────────┐
   CLI              │   SHARED CORE: shield/       │   Web app
   protect.py ─────►│                             │◄──── backend/main.py
   robustness.py ──►│  clip_model.py  attack.py   │      (FastAPI)
   eval_transfer ──►│  perceptual.py  metrics.py  │            ▲
                    │  transforms.py  diff_trans  │            │
                    │  pipeline.py (glue)         │      frontend/ (HTML/JS)
                    └─────────────────────────────┘
```

- **CLI**: for technical users — run experiments, tune params (`--epsilon`, `--steps`, `--eot`...).
- **Web app**: for artists — drag-drop, pick a preset.
- Both call into `shield/` — especially `pipeline.py`, the "glue" of shared logic for the web.

## 2. Layers and responsibilities

| Layer | File | Responsibility |
|-------|------|----------------|
| **Model** | `shield/clip_model.py` | Wrap CLIP: encode image/text (keep pixel gradients), classify |
| **Algorithm** | `shield/attack.py` | PGD, PGD-EoT, ensemble — craft noise |
| **Transforms (train)** | `shield/diff_transforms.py` | Differentiable blur/resize/compress for EoT |
| **Transforms (test)** | `shield/transforms.py` | PIL blur/resize/compress to measure robustness |
| **Perceptual** | `shield/perceptual.py` | Masking map to hide noise |
| **Metrics** | `shield/metrics.py` | PSNR, SSIM |
| **Orchestration** | `shield/pipeline.py` | Assemble into `protect_image()` + `run_robustness()` |
| **Web I/O** | `backend/main.py` | FastAPI: `/api/protect`, `/api/verify`, static files |
| **Config** | `backend/config.py` | Light/Medium/Strong presets, paths, limits |
| **UI** | `frontend/` | Drag-drop, show results, call API |

> Core design principle: **separate the pure algorithm (shield/) from I/O (CLI / web)**. Changing the UI doesn't touch the algorithm, and vice versa.

## 3. Data flow — "Protect an image"

The most important flow. Example via the web:

```
1. User drops an image + picks "medium" → frontend POSTs /api/protect
2. backend/main.py:
   - _load_and_fit(): read image, shrink if longest side > MAX_IMAGE_SIDE (512)
   - look up PRESETS["vua"] → {epsilon, steps, mask_min, eot_samples}
   - call pipeline.protect_image(...)
3. pipeline.protect_image():
   - get_clip(): load CLIP (lazy singleton — once only)
   - pil_to_tensor(): PIL image → tensor (1,3,H,W) in [0,1]
   - perceptual_mask(): build the masking map
   - pgd_protect_eot(): THE OPTIMIZATION LOOP (heaviest part, ~1-2 min CPU)
   - compute cosine to original, PSNR, SSIM → metrics dict
4. backend saves {uid}_original.png + {uid}_protected.png into storage/web/
5. returns JSON: {uid, metrics, original_url, protected_url}
6. frontend shows before/after images + metrics
```

## 4. Data flow — "Verify robustness"

```
1. User clicks "Check robustness" → POST /api/verify with uid
2. backend reads back the two saved images (original, protected)
3. pipeline.run_robustness():
   - compute the original embedding as reference
   - for each transform in build_default_suite() (JPEG 90/70/50, resize, blur, median, quantize, combo):
       • apply it to the PROTECTED image
       • compute cosine to the original
       • "still protected" if cosine < threshold (0.5)
   - count survivors (held/total)
4. return a result table → frontend renders it
```

## 5. Why split `diff_transforms.py` and `transforms.py`?

A subtle architectural decision, often asked:

| | `diff_transforms.py` | `transforms.py` |
|--|---------------------|-----------------|
| Tooling | torch (tensors) | PIL (images) |
| Differentiable? | **Yes** (gradients flow) | No |
| Used when | **TRAINING** the noise (EoT) | **TESTING** robustness |
| Why | EoT needs gradients to optimize through transforms | Tests just simulate reality (real JPEG, real resize) |

Using one for both would be wrong: PIL when training → no gradients; torch when testing → doesn't reflect the real JPEG a scraper uses.

## 6. Lazy singleton — load CLIP once

CLIP is heavy (~350MB) and takes seconds to load. `pipeline.py` uses a **lazy singleton**:
```python
_clip = None
def get_clip():
    global _clip
    if _clip is None:
        _clip = ClipWrapper()   # load only the first time
    return _clip
```
So: the first request is slow (load), later requests are fast (reuse). Good for a long-running server.

## 7. How the backend serves the frontend

`backend/main.py` mounts two static dirs, **in the right order**:
```python
app.mount("/files", StaticFiles(directory=WEB_STORAGE_DIR))   # processed images
app.mount("/", StaticFiles(directory=FRONTEND_DIR, html=True)) # the UI
```
> ⚠️ Mounting `/` must come **last**, after `/api/*` and `/files`. If placed first, it would "swallow" every request and hide the APIs. A common FastAPI StaticFiles pitfall.

## 8. Size limit so CPU keeps up

The PGD-EoT loop is heavy (each step encodes the image through CLIP many times). To make it usable on CPU:
- `MAX_IMAGE_SIDE = 512`: large images shrink to 512px max.
- Presets tune `steps` and `eot_samples` to balance speed/quality (Light ~1 min, Strong ~3 min).

## 9. Module dependency sketch

```
protect.py ───────┐
robustness.py ────┤
eval_transfer.py ─┼──► shield/attack.py ──► shield/clip_model.py ──► transformers (CLIP)
                  │         │                                   └──► torch
backend/main.py ──┘         └──► shield/diff_transforms.py ──► torch
       │
       └──► shield/pipeline.py ──► attack + clip_model + perceptual + metrics + transforms
              │
backend/config.py (PRESETS, paths)
```

`pipeline.py` is the hub: it imports nearly every module under `shield/` and is the only thing `backend` needs to know.
