# 06 — Standout Features (your point-scorers)

These are the most notable technical points to emphasize — with phrasing.

---

## 1. Hand-rolled PGD + EoT in PyTorch ⭐⭐⭐

**Why it stands out**: no off-the-shelf adversarial library (foolbox, ART) — you wrote the gradient loop, projection, and EoT yourself. Shows deep understanding.

**How to say it**: "I optimize a noise tensor `delta` via the gradient of cosine similarity. Each step I `backward` the similarity, step against the gradient sign to push the embedding away, then project `delta` into `[−ε, ε]` and keep the image in `[0,1]`."

## 2. EoT — turning "useless" noise into "9/9 robust" ⭐⭐⭐

**Why it stands out**: the nicest *improvement story* in the project. Naive PGD survives 0/9 wash-off transforms; with EoT it reaches 9/9. With before/after numbers.

**How to say it**: "Naive PGD noise lives in high frequencies, which compression/resize erase, so it vanishes. I applied EoT: each optimization step I apply several random differentiable transforms and average the gradient, forcing the noise to survive *while being* blurred/resized/compressed. Robustness jumped from 0/9 to 9/9."

## 3. STE — gradients through rounding ⭐⭐

**Why it stands out**: a subtle, lesser-known detail. Simulating compression needs color rounding, which has zero gradient.

**How to say it**: "To simulate compression in EoT I need color quantization, but `round` blocks gradients. I used a straight-through estimator: round on the forward pass, identity on the backward — via `x + (q - x).detach()`."

## 4. Perceptual masking — hide noise in detail ⭐⭐

**Why it stands out**: combines **human vision** knowledge (contrast masking) with technique. With numbers: a real image reaches **SSIM 0.94** (noise nearly invisible) while staying 9/9 robust.

**How to say it**: "Humans easily see noise in flat areas like sky, but barely in detailed areas. I compute the local std of luminance as a mask and allocate the noise budget by it — strong noise in detail, suppressed in smooth regions."

## 5. Scientific honesty: measuring transferability ⭐⭐⭐

**Why it stands out**: `eval_transfer.py` measures transfer between two CLIP variants: B/32-targeted noise has little effect on B/16 (cosine +0.84). **GPT-4o and generative-model training were not evaluated**.

**How to say it**: "I don't sugarcoat. I wrote an experiment comparing: noise targeting one model only protects on that model and is nearly useless on a different architecture. That's why no tool — including Glaze — is 100% immune. I treat it as a friction layer that raises the cost for abusers, combined with watermarks and terms of use."

> 💡 Honesty about limits **scores more** than overclaiming. Interviewers value critical thinking.

## 6. Ensemble attack to improve transfer ⭐

**How to say it**: "To improve transfer I have `pgd_protect_eot_multi`, which optimizes the noise across multiple CLIPs at once, forcing it to exploit *shared* features rather than one model's quirks. It covers that model set well, though it still can't guarantee a totally unseen model."

## 7. The right evaluation metric (cosine, not label flip) ⭐⭐

**Why it stands out**: shows precise measurement thinking.

**How to say it**: "It's tempting to think 'flipping the classification label means success'. But the label can stay the same while the embedding has moved far. So I use **cosine similarity to the original** as the main metric (threshold < 0.5); label flips are just secondary."

## 8. Splitting train (differentiable) / test (real) transforms ⭐

**How to say it**: "I train EoT with *differentiable* torch transforms to get gradients, but I test robustness with PIL's *real* JPEG/resize for objective results — no fooling myself with an approximation."

## 9. Shared core for CLI + Web ⭐

**How to say it**: "`shield/pipeline.py` is the single core; both the CLI and FastAPI call it. Changing the UI doesn't touch the algorithm."

---

## "What to emphasize by JD" table

| JD emphasizes | Emphasize this |
|---------------|----------------|
| AI/ML, deep learning | Hand-rolled PGD/EoT, gradients, STE, embeddings/cosine |
| Computer vision | CLIP, perceptual masking, PSNR/SSIM |
| Research / R&D | transferability, ensemble, honesty about limits |
| Backend/Fullstack | FastAPI, shared pipeline, presets, drag-drop web |
| Security / adversarial ML | threat model, robustness, reversed defender/attacker roles |
