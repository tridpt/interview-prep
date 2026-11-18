# 04 — Core Internals (function by function)

This walks through **every important function** in `shield/`. This is the "guts" — get this and you own the project.

---

## A. `clip_model.py` — Wrapping CLIP

### Design goal
Most important: `encode_image` takes an image as a **pixel tensor in [0,1]** and **normalizes internally**. This lets **gradients flow back to each pixel** — *required* for crafting adversarial noise.

### `__init__`
```python
self.model = CLIPModel.from_pretrained(model_name).to(device).eval()
for p in self.model.parameters():
    p.requires_grad_(False)   # DON'T learn model weights, only optimize the noise
```
- `.eval()`: disable dropout/batchnorm updates.
- **Freeze parameters**: we don't train CLIP; we optimize `delta` (the noise). Freezing saves memory and ensures the gradient is taken only w.r.t. the input image.
- Stores CLIP's `mean`/`std` as `(1,3,1,1)` tensors to broadcast during normalization.

### `preprocess_pixels`
```python
pixels = F.interpolate(pixels, size=(224,224), mode="bicubic")  # resize to 224
pixels = pixels.clamp(0,1)
return (pixels - self._mean) / self._std                        # normalize
```
- CLIP needs 224×224, normalized with OpenAI's mean/std.
- Uses `F.interpolate` (differentiable), **not** PIL resize → gradients flow through resize.

### `_coerce` — transformers version compatibility
`get_image_features` returns a raw tensor in old versions but an object with `image_embeds`/`pooler_output` in transformers 5.x. This function probes attributes to always retrieve the embedding vector. → A defensive detail showing the author cares about compatibility.

### `encode_image`
```python
normalized = self.preprocess_pixels(pixels)
out = self.model.get_image_features(pixel_values=normalized)
return F.normalize(self._coerce(out), dim=-1)   # L2-normalize (length = 1)
```
L2-normalize so cosine similarity is a simple dot product.

### `encode_text` and `classify`
- `encode_text`: tokenize captions → text embeddings (under `@torch.no_grad()` since text needs no gradient).
- `classify`: `logits = (img @ txt.T) * logit_scale.exp()`, then `softmax` → per-label probability, sorted descending. This is CLIP zero-shot classification. **Note**: in this project, the label is only *secondary* info; the main metric is cosine.

---

## B. `attack.py` — The heart of the project

### `pgd_protect` — basic PGD
```python
with torch.no_grad():
    target_emb = clip.encode_image(image).detach()  # the reference to move away from

delta = torch.empty_like(image).uniform_(-epsilon, epsilon)  # random noise
delta.requires_grad_(True)

for i in range(steps):
    adv = (image + delta).clamp(0, 1)
    adv_emb = clip.encode_image(adv)
    similarity = (adv_emb * target_emb).sum(dim=-1).mean()   # cosine to original

    similarity.backward()                  # gradient of similarity w.r.t. delta
    with torch.no_grad():
        delta -= step_size * delta.grad.sign()   # GO AGAINST to DECREASE similarity
        delta.clamp_(-epsilon, epsilon)          # project to [−ε, ε]
        if mask is not None:                     # hide noise via mask
            bound = epsilon * mask
            delta.copy_(torch.clamp(delta, -bound, bound))
        delta.copy_((image + delta).clamp(0,1) - image)  # keep image in [0,1]
```

Points to explain in an interview:
- **`target_emb` is `.detach()`-ed**: it's a constant (reference), no gradient needed.
- **`delta -= step_size * sign(grad)`**: we want to **decrease** similarity (push away), so go *against* the gradient of similarity. `sign()` (PGD/FGSM-style) gives stable steps.
- **Three projections per step**: (1) clip to `[−ε,ε]`; (2) if a mask exists, clip to `ε*mask`; (3) ensure `image+delta ∈ [0,1]` by adjusting `delta`. This order respects all constraints at the end of each step.
- `step_size` defaults to `2.5 * epsilon / steps` — a common PGD rule of thumb.

### `pgd_protect_eot` — robust PGD (the biggest highlight)
Differs from `pgd_protect` in **how similarity is computed** each step:
```python
sims = []
for _ in range(eot_samples):
    transformed = sample_transform(adv)        # RANDOM (differentiable) transform
    emb = clip.encode_image(transformed)
    sims.append((emb * target_emb).sum(-1).mean())
emb_clean = clip.encode_image(adv)             # also include the NO-transform case
sims.append((emb_clean * target_emb).sum(-1).mean())
similarity = torch.stack(sims).mean()          # EXPECTATION (average)
```
- Average similarity over `eot_samples` random transforms + one clean pass.
- Optimizing this average → the noise must survive *every* transform → robust.
- Why add the clean case (`emb_clean`)? So the noise still fools the **untransformed** image, not only transformed ones.

### `pgd_protect_eot_multi` — model ensemble
Like EoT but the loop iterates over **multiple CLIPs** (`clips`), accumulating similarity over each:
```python
for c, tgt in zip(clips, targets):
    for _ in range(eot_samples):
        emb = c.encode_image(sample_transform(adv))
        sims.append((emb * tgt).sum(-1).mean())
    sims.append((c.encode_image(adv) * tgt).sum(-1).mean())
```
- Goal: the noise must corrupt **every** model in the set → exploit *shared* features → better transfer to unseen models.
- Used in `eval_transfer.py` to compare (A) targeting only B/32 vs (B) ensemble B/32+B/16.

---

## C. `diff_transforms.py` — DIFFERENTIABLE transforms (for EoT)

All in torch so gradients flow.

| Function | What | Simulates |
|----------|------|-----------|
| `diff_gaussian_blur` | Conv with a Gaussian kernel from sigma | Blur / denoise |
| `diff_resize_roundtrip` | Downscale then upscale via `F.interpolate` | Scraper resize |
| `ste_quantize` | Color quantization via **STE** | Compression / bit reduction |
| `add_noise` | Add Gaussian noise | Denoise/compression distortion |

### `sample_transform` — the EoT sampler
```python
if random.random() < 0.8: out = diff_resize_roundtrip(out, scale=uniform(0.4, 0.85))
if random.random() < 0.8: out = diff_gaussian_blur(out, sigma=uniform(0.5, 1.5))
if random.random() < 0.6: out = ste_quantize(out, bits=choice([4,5,6]))
if random.random() < 0.5: out = add_noise(out, std=uniform(0.005, 0.02))
return out.clamp(0, 1)
```
Each transform is **toggled randomly** with random params → simulates "we don't know how the scraper will process the image". This diversity is the essence of EoT.

### `ste_quantize` — explained
```python
levels = 2**bits - 1
q = torch.round(x.clamp(0,1) * levels) / levels
return x + (q - x).detach()
```
Forward yields `q` (rounded); backward: since `(q-x)` is `.detach()`-ed, the derivative is `d/dx = 1`. This "tricks" autograd into passing gradients through the non-differentiable rounding.

---

## D. `transforms.py` — TEST transforms (PIL, non-differentiable)

Simulate exactly what scrapers do in reality:
- `jpeg_compress`: save JPEG at a given quality then reopen (**real** compression, not an approximation).
- `resize_roundtrip`: downscale/upscale via PIL BICUBIC.
- `gaussian_blur`, `median_filter`, `bit_quantize`.
- `build_default_suite()`: returns 9 transforms — JPEG 90/70/50, resize 50%/75%, blur r=1.0, median 3×3, 4-bit color, plus one harsh **combo** (resize 50% + JPEG 70).

> The pair `diff_transforms` (train) ↔ `transforms` (test) is a nice design: train on a *differentiable approximation*, but test on the *real* thing for objective results.

---

## E. `perceptual.py` — Perceptual masking map

```python
lum = 0.299*r + 0.587*g + 0.114*b          # perceived luminance
mean = avg_pool2d(lum_padded, window)       # local mean
mean_sq = avg_pool2d(lum_padded**2, window)
std = sqrt(mean_sq - mean**2)               # LOCAL STD = "detail richness"
ref = quantile(std, 0.9)                     # normalization anchor (robust to outliers)
detail = (std / ref).clamp(0, 1)
mask = m_min + (1 - m_min) * detail          # squeeze into [m_min, 1]
```
- **High local std** = many edges/detail = allow strong noise.
- **Low std** = flat region = suppress noise toward `m_min` (keep smooth areas clean).
- Normalize by the **0.9 percentile** instead of max → robust to a few extreme pixels.
- Returns a `(1,1,H,W)` mask to multiply the noise budget in `attack.py`.

---

## F. `metrics.py` — Quality measurement

- `to_uint8`: convert `[0,1]` float → `uint8 [0,255]`.
- `psnr`: `20*log10(255/sqrt(MSE))`. If MSE=0 → returns `inf` (identical).
- `ssim`: calls `skimage.metrics.structural_similarity`; if scikit-image is missing, returns `None` (soft fallback, app still runs).

---

## G. `pipeline.py` — The glue

- `get_clip()`: lazy singleton (see file 03, §6).
- `protect_image(img, epsilon, steps, use_eot, use_perceptual, mask_min, eot_samples)`: does the whole protection and returns `(PIL image, metrics dict)`. This is what the web calls.
- `run_robustness(original, protected, threshold)`: runs the test suite, returns `{baseline_cosine, rows, held, total}`.
- Constants `PROTECT_THRESHOLD = 0.5` and `LABELS` live here.

> In an interview, if asked "the flow from image in to result out", narrate `protect_image`: pil_to_tensor → perceptual_mask → pgd_protect_eot → measure cosine/PSNR/SSIM → tensor_to_pil.
