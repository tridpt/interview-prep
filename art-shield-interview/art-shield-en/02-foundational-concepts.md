# 02 — Foundational Concepts (from zero)

This file gives you all the theory needed to understand Art Shield. Read it carefully and the rest will be easy.

---

## 1. What is an embedding (feature vector)?

Computers don't "understand" images like we do. An AI model turns an image into a **list of numbers** — an **embedding** (feature vector). For example, CLIP turns each image into a **512-dimensional** vector.

Meaning: two images that are **similar in content** have **nearby** embeddings in the 512-D space; different images have **far** embeddings. The embedding is "how the AI sees the image".

> 🔑 The core of Art Shield: **make the protected image's embedding sit far from the original's**. Then the AI "sees" something else entirely, even though the image looks unchanged.

## 2. Cosine similarity

How do we measure "near/far" between two vectors? **Cosine similarity** — the cosine of the angle between them:

- **1.0**: same direction → AI sees "identical".
- **0**: orthogonal → unrelated.
- Negative (e.g. **−0.4**): opposite direction → AI sees "the opposite".

In Art Shield, embeddings are **L2-normalized** (length = 1), so cosine is just a dot product: `(a * b).sum()`.

> The project's "still protected" criterion: **cosine to original < 0.5**. Lower is better.

## 3. What is CLIP?

**CLIP** (Contrastive Language–Image Pre-training, OpenAI 2021) learns to **link images with text**. It has two encoders:

- **Image encoder**: image → 512-D vector.
- **Text encoder**: a caption → 512-D vector.

CLIP is trained so an image and its correct caption have nearby embeddings. This enables **zero-shot classification**: given an image and a list of captions ("a landscape painting", "a portrait"...), CLIP computes cosine between the image and each caption; the highest is the predicted label.

**Why target CLIP?** Because CLIP (and similar image encoders) is the "eye" of many AI image-generation and data-filtering systems. Corrupting a painting's CLIP features makes it hard to use in those pipelines.

Art Shield uses `openai/clip-vit-base-patch32` (B/32) by default.

## 4. What is an adversarial perturbation?

A classic finding in deep learning: neural nets are **fragile**. Adding a **tiny, purposeful** perturbation (not random noise) can make a model predict completely wrong — while humans see no difference.

Why: a model is a complex function with millions of parameters. In the high-dimensional image space there are "sensitive directions" where nudging pixels just slightly along them changes the output a lot. Adversarial noise hunts for exactly those directions.

**How to find them: gradients.** A gradient tells you "which direction to change pixels to change the objective fastest". We follow the gradient to **decrease** the cosine similarity to the original.

## 5. PGD (Projected Gradient Descent)

PGD is a common, strong way to craft adversarial noise. The procedure (this is `pgd_protect`):

1. Compute the original image's embedding → `target_emb` (the target to move away from).
2. Init a small random `delta` in `[−ε, ε]`.
3. Loop `steps` times:
   - Build `adv = clamp(image + delta, 0, 1)`.
   - Compute `similarity = cosine(encode(adv), target_emb)`.
   - Backprop to get the gradient of similarity w.r.t. `delta`.
   - **Update**: `delta -= step_size * sign(gradient)` — go against the sign to **decrease** similarity (push away).
   - **Project**: clip `delta` back into `[−ε, ε]`, and keep `image + delta` in `[0,1]`.

Two key terms:
- **ε (epsilon)**: the max noise budget under the **L-infinity** norm (each pixel can change by at most ε). Small ε = invisible but weak; large ε = strong but visible.
- **sign(gradient)**: PGD uses the *sign* of the gradient (FGSM-style), not its magnitude — steady, stable steps.

## 6. The fatal problem: naive noise is easily "washed off"

PGD noise tends to live in **high frequencies** (tiny pixel changes). But everyday operations — **JPEG compression, resize, blur** — are smoothing filters that **erase high frequencies**. So a scraper just resizes the image and the noise vanishes; the AI sees the original again.

Per the README experiment: naive PGD (ε=0.03) survives **0/9** transforms. Useless in the real world.

## 7. EoT (Expectation over Transformation) — robust noise

The fix: **force the noise to survive *while being* transformed**. This is EoT (`pgd_protect_eot`).

Idea: each PGD step, instead of computing the gradient on the "clean" image, we:
1. Apply **several random transforms** (blur, resize, compress, noise) to `adv`.
2. Compute similarity on **each** transformed image, then **average** (the "expectation").
3. Optimize the noise based on that average.

Result: the noise must exploit **robust features** — ones that survive processing. Experiment: EoT reaches **9/9** survival.

> 🔑 Key requirement: the transforms used in EoT must be **differentiable** so gradients can flow through. That's why there are two separate files: `diff_transforms.py` (differentiable, torch — used WHEN TRAINING the noise) and `transforms.py` (PIL, non-differentiable — used only to TEST).

## 8. STE (Straight-Through Estimator) — a trick for rounding

To simulate compression, we need to **quantize colors** (round pixel values). But rounding has a gradient of 0 everywhere → no gradient flows, EoT breaks.

The **STE** trick: round for real on the forward pass; treat it as identity (gradient = 1) on the backward pass. In code:
```python
q = torch.round(x * levels) / levels   # rounded value
return x + (q - x).detach()            # forward = q, backward flows through x
```
`.detach()` cuts the gradient of `(q - x)`, so the derivative is only w.r.t. `x`.

## 9. Perceptual masking — hide the noise

Humans are **not equally sensitive** to noise: we easily see it in **flat, smooth regions** (sky, plain walls) but barely in **detailed/edge regions** (leaves, hair, texture). This is *contrast masking*.

`perceptual.py` exploits this: it computes the **local standard deviation** of luminance (detail-rich areas have high std) → builds a **mask** valued in `[m_min, 1]`. Detailed areas get strong noise (near 1), flat areas suppress it (near `m_min`). This yields higher SSIM (more invisible) while staying robust.

## 10. PSNR and SSIM — "did the image change to a human?"

To prove the noise is hard to see, the project uses two metrics:

- **PSNR** (Peak Signal-to-Noise Ratio, dB): compares pixel differences. **Higher** = more similar. > 40 dB = barely noticeable; ~30 dB is still quite hidden.
- **SSIM** (Structural Similarity Index, 0..1): models how humans perceive structure/contrast. Closer to **1** = more similar. SSIM 0.94 = noise nearly invisible.

## 11. Transferability — does the noise carry to other models?

The crucial question: does noise targeting CLIP-B/32 corrupt **other models** (B/16, GPT-4o...)? This is **transferability**.

In `eval_transfer.py`, transfer between **two CLIP models** is poor: B/32-targeted noise changes its own embedding (cosine −0.49) but has less effect on B/16 (cosine +0.84). **GPT-4o was not tested**; do not extrapolate a measured result to it.

**Improvement**: attack an **ensemble** of models at once (`pgd_protect_eot_multi`) → the noise must exploit shared features, somewhat improving transfer. But it can **never** cover every model, especially closed ones.

## 12. Threat model — who is the "attacker"?

Here the roles are **reversed** vs a normal adversarial attack:
- "Defender" = the **artist** (crafts noise to protect).
- "Attacker" = the **data scraper** (tries to wash off the noise to train AI).

The artist wins if the noise survives the washing. This is **adversarial machine learning** applied to copyright protection.

---

## Quick glossary

| Term | One line |
|------|----------|
| Embedding | Numeric vector describing image content (CLIP: 512-D) |
| Cosine similarity | Direction similarity of two vectors (1=same, <0.5=protected) |
| CLIP | Image–text model, the "AI eye" we fool |
| Adversarial perturbation | Tiny purposeful noise that makes AI misread |
| PGD | Iterative gradient + projection to craft noise |
| ε (epsilon) | Max noise budget (L-inf norm) |
| EoT | Optimize noise over many transforms → robust |
| STE | Lets gradient pass through rounding |
| Perceptual mask | Hide noise in detailed regions |
| PSNR / SSIM | Measure image similarity to humans |
| Transferability | Noise carrying to other models (here: poor) |
