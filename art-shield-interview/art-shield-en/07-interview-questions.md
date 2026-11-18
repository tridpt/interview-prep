# 07 — Interview Questions (with model answers)

Ordered basic → advanced. Each has a short answer to rephrase in your own words.

---

## A. Overview & motivation (1–10)

**1. What does Art Shield do in one sentence?**
Adds a tiny, human-invisible perturbation to a painting so an AI model (CLIP) extracts wrong features — making the art hard to use for AI training.

**2. The real problem it solves?**
Artists' work is scraped en masse to train generative models without consent; Art Shield raises the cost/difficulty of that.

**3. What real tools is it like?**
Glaze and Nightshade from the University of Chicago — same adversarial-noise idea to protect artwork.

**4. Why target CLIP and not another model?**
CLIP is a common image encoder, the "eye" of many AI image/data pipelines. Corrupting CLIP features makes the art hard to use there.

**5. Main technologies?**
PyTorch (gradients), CLIP via transformers, hand-rolled PGD/EoT, PSNR/SSIM (scikit-image), FastAPI + vanilla JS for the web.

**6. How many stages?**
3: (1) prove the technique, (2) measure & upgrade robustness (EoT, masking), (3) web app.

**7. System input/output?**
In: an image + protection level. Out: the perturbed image + metrics (cosine to original, PSNR, SSIM) + optional robustness report.

**8. Who are the users?**
Artists/creators wanting to protect their work before posting online.

**9. Why both CLI and web?**
CLI for technical users to tune params/run experiments; web for non-technical artists (drag-drop, presets).

**10. Biggest limitation?**
Not absolute protection; noise targeting one model **doesn't transfer** well to a different architecture; can weaken under heavy processing.

---

## B. Foundational concepts (11–28)

**11. What is an embedding?**
A numeric vector describing image content (CLIP: 512-D). Similar images → nearby embeddings.

**12. What does cosine similarity measure? Why use it?**
The cosine of the angle between two vectors. Since embeddings are L2-normalized, cosine = dot product, measuring "how similar the AI sees two images". Used as the main protection metric.

**13. Why is the threshold cosine < 0.5?**
Below 0.5 means the protected embedding moved far enough from the original → the AI "sees" something else → protection succeeded.

**14. Adversarial vs random noise?**
Adversarial noise is small but **purposeful** (found via gradients) and makes the model mispredict; random noise is usually harmless because it doesn't target sensitive directions.

**15. Why does tiny noise cause big errors?**
Neural nets have "sensitive directions" in high-dimensional space; nudging pixels along them changes the output a lot while humans don't notice.

**16. What is PGD? Describe the loop.**
Projected Gradient Descent: iterate; each step move along the gradient sign to decrease similarity, then project the noise to `[−ε,ε]` and keep the image in `[0,1]`.

**17. What is ε? The trade-off?**
Max per-pixel noise budget (L-inf norm). Large = strong but visible; small = hidden but weak.

**18. Why `sign(gradient)` instead of the full gradient?**
PGD/FGSM-style: the sign gives steady, stable steps unaffected by fluctuating gradient magnitude.

**19. What does "project" mean in PGD?**
After each step, clip the noise back to the valid set `[−ε,ε]` and ensure image+noise ∈ `[0,1]`. Keeps constraints satisfied.

**20. Why is naive PGD noise easily washed off?**
It lives in high frequencies; compression/resize/blur are smoothing filters that erase high frequencies → the noise disappears.

**21. What is EoT?**
Expectation over Transformation: optimize the noise across many random transforms and average, forcing it to survive image processing.

**22. Why must EoT transforms be differentiable?**
So gradients flow through the transform to optimize the noise; non-differentiable transforms give no gradient.

**23. What is STE? Where is it used?**
Straight-through estimator: lets gradients pass through rounding (forward rounds, backward is identity). Used in `ste_quantize` to simulate compression.

**24. What is perceptual masking?**
Hiding noise in detailed areas (hard to see) and suppressing it in flat areas. Based on local std of luminance.

**25. PSNR vs SSIM?**
PSNR measures pixel differences (dB, higher = more similar); SSIM models human perception of structure (0..1, closer to 1 = more similar). SSIM is closer to human perception.

**26. How does CLIP do zero-shot classification?**
Cosine between the image embedding and each caption embedding, softmax → the highest-cosine label.

**27. What is transferability? The project's result?**
Whether noise carries to other models. The experiment compares only CLIP B/32 and B/16 and shows poor transfer; **GPT-4o was not tested**.

**28. The threat model?**
Defender = artist (crafts noise); attacker = scraper (washes it off). The artist wins if the noise survives.

---

## C. Implementation & code (29–48)

**29. Why freeze CLIP's parameters?**
We don't train the model, only the noise. Freezing saves memory and ensures the gradient is only w.r.t. the image.

**30. Why does `encode_image` take a [0,1] tensor and normalize internally?**
So gradients flow to the raw pixels — required to optimize the noise. Normalization stays in the computation graph.

**31. Why `F.interpolate` resize instead of PIL?**
`F.interpolate` is differentiable so gradients pass; PIL resize cuts gradients.

**32. Why is `target_emb` `.detach()`-ed?**
It's a fixed reference (the original embedding) needing no gradient; detaching stops autograd from tracing back.

**33. In PGD, why `delta -= step_size * grad.sign()` (minus)?**
We want to **decrease** similarity (push away), so move against the similarity gradient.

**34. What do the three projections at the end of each step do?**
(1) clip to `[−ε,ε]`; (2) if a mask exists, clip to `ε*mask`; (3) adjust delta so `image+delta ∈ [0,1]`.

**35. How does `sample_transform` work?**
Randomly toggles 4 transforms (resize, blur, quantize, noise) with random params — one diverse EoT sample.

**36. Why does EoT add the "clean" case?**
So the noise still fools the untransformed image, not only transformed ones.

**37. How is `pgd_protect_eot_multi` different? Its purpose?**
Optimizes the noise across multiple CLIPs → exploits shared features → better transfer to unseen models.

**38. How does `perceptual_mask` compute "detail"?**
Local std of luminance via a sliding window (avg_pool of x² minus the square of avg_pool of x).

**39. Why normalize the mask by the 0.9 percentile instead of max?**
To be robust to a few extreme pixels (outliers) skewing the scale.

**40. `to_uint8` and its role in metrics?**
Converts [0,1] float → uint8 [0,255] so PSNR/SSIM are computed on the same scale as the saved real images.

**41. Why split `diff_transforms.py` and `transforms.py`?**
Training needs differentiable (torch) for gradients; testing needs real ops (PIL/JPEG) for objectivity.

**42. What pattern is `pipeline.get_clip()`? Benefit?**
Lazy singleton: load CLIP once, reuse for all requests → first request slow, later ones fast.

**43. What does `protect_image` return?**
`(protected PIL image, metrics dict)` with cosine, protected (bool), psnr, ssim, before/after labels.

**44. What does `run_robustness` return?**
`{baseline_cosine, threshold, rows[], held, total}` — per-transform table and survivor count.

**45. Why mount `/` last in FastAPI?**
So the StaticFiles `/` mount doesn't hide the `/api/*` and `/files` routes.

**46. Why insert `BASE_DIR` into `sys.path` in the backend?**
To import the root-level `shield/` package from inside `backend/`.

**47. What's in a preset? Why preset-ize?**
`epsilon, steps, mask_min, eot_samples` for Light/Medium/Strong. Presets so end users needn't understand technical params.

**48. Why append `?t=Date.now()` to image src on the frontend?**
Cache busting so the latest processed image always shows.

---

## D. Deep / challenge questions (49–62)

**49. How do you know the noise is "invisible"? Measured how?**
PSNR/SSIM; e.g. SSIM 0.94 on a real image means very small structural difference.

**50. What if the attacker uses a model you don't know?**
The noise may be useless (poor transfer). Ensemble helps somewhat but can't cover all — this is the core limitation, and I say so directly.

**51. Why not use an existing adversarial library?**
To understand the mechanics deeply and have full control (custom mask, EoT). The project's goal is learning/research.

**52. Trade-off of large ε?**
Stronger protection but more visible noise (lower SSIM/PSNR). The "Strong" preset accepts more visibility for more robustness.

**53. Why measure with cosine rather than "AI guesses the wrong label"?**
The label can stay the same while the embedding moved far; cosine reflects "the AI sees it differently" more faithfully.

**54. Does EoT slow things down? Why?**
Yes — each step encodes the image `eot_samples+1` times through CLIP. Hence the 512px limit and preset-tuned `eot_samples`.

**55. What changes on a GPU?**
Much faster (CLIP on CUDA); could raise steps/eot_samples and process larger images.

**56. Ethical risk of the tool?**
It's defensive (protect work), but the same adversarial technique can attack. The project positions itself clearly as protection + learning.

**57. How to extend to a closed model (GPT-4o)?**
No gradient of a closed model → can't optimize directly. Only transfer (poor) applies. This is the fundamental black-box barrier.

**58. Next improvements you'd consider?**
Ensemble over more model families, frequency-space optimization, or combining with watermarks; add auth if deployed for real.

**59. Why does SSIM sometimes return None?**
When scikit-image is missing — a soft fallback so the app still runs, just without SSIM.

**60. Biggest lesson from the project?**
Adversarial noise looks great on paper but is useless if not robust; EoT is mandatory; and one must be honest that there's no absolute shield.

**61. What if you had to protect 10,000 images?**
Need a GPU and batch processing; the current CPU+web is for one-offs. Could move to Colab/GPU server.

**62. What does a negative cosine (e.g. −0.42) mean?**
The protected embedding is nearly **opposite** to the original — the AI sees opposing content, very strong protection.

---

## Answering tips
- Always connect **theory to real code** ("I `.detach()` `target_emb` because...").
- When asked about limits, **admit them directly** then offer improvements — don't be defensive.
- Keep a few **numbers** ready (0/9 → 9/9, SSIM 0.94, cosine −0.49 vs +0.84) for persuasiveness.
