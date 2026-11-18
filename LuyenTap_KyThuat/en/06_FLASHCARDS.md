# 6. Flashcards — Quick Q&A review

Cover the answer, answer it yourself. Skim before go-time. (Q = question, A = short answer.)

## JavaScript
- **Q:** `let` vs `const` vs `var`? **A:** block / block-no-reassign / function-avoid.
- **Q:** `==` vs `===`? **A:** coercion vs strict; use `===`.
- **Q:** Closure? **A:** a function remembering outer-scope variables.
- **Q:** Event loop? **A:** single thread + callback/microtask queues for I/O.
- **Q:** Promise's 3 states? **A:** pending / fulfilled / rejected.
- **Q:** localStorage vs IndexedDB? **A:** text/sync/5MB vs object/async/large.
- **Q:** Read a stream? **A:** `response.body.getReader()` + read chunks.
- **Q:** Event delegation? **A:** one listener on a container + `closest()`.

## Python
- **Q:** list/tuple/set/dict? **A:** mutable-ordered / immutable / dedup / key-value.
- **Q:** When async? **A:** I/O-bound (network/file).
- **Q:** Async generator? **A:** `async def` + `yield` → stream.
- **Q:** GIL? **A:** one thread runs bytecode → threading suits I/O, CPU uses multiprocessing.
- **Q:** Decorator? **A:** a function wrapping another (`@app.post`, `@dataclass`).
- **Q:** What's a dataclass for? **A:** compact data struct, auto `__init__`.
- **Q:** Pydantic? **A:** validate requests against a schema, 422 on error.

## Web / Real-time
- **Q:** 401 vs 403? **A:** unauthenticated vs no permission.
- **Q:** 429? **A:** too many requests (rate limit).
- **Q:** SSE vs WebSocket? **A:** one-way vs two-way.
- **Q:** PWA needs? **A:** HTTPS + manifest + service worker.
- **Q:** SW lifecycle? **A:** install → activate → fetch.
- **Q:** Cache strategies? **A:** cache-first / network-first / stale-while-revalidate.
- **Q:** REST stateless meaning? **A:** each request is self-contained.
- **Q:** CORS? **A:** controls cross-origin API calls.

## Security / Crypto (tied to totp-manager, quan-quen-map)
- **Q:** How to prevent XSS? **A:** escape dynamic content before innerHTML.
- **Q:** Symmetric vs asymmetric encryption? **A:** same key / public-private key pair.
- **Q:** Hash vs encryption? **A:** one-way (no decrypt) vs two-way (decryptable).
- **Q:** What's a KDF (scrypt/PBKDF2) for? **A:** password → key, deliberately slow to resist brute-force.
- **Q:** Salt? **A:** random string defeating lookup tables (rainbow tables).
- **Q:** AES-GCM? **A:** symmetric encryption that also authenticates integrity.
- **Q:** TOTP? **A:** time-based OTP: HMAC(secret, time//30) + truncation.
- **Q:** "No stored key" meaning? **A:** key only in RAM; forget password = lose data.

## AI/ML & Speech (tied to art-shield, token-counter, voice-clone-web)
- **Q:** What is an embedding? **A:** a numeric vector describing content (CLIP: 512-D); similar → close.
- **Q:** Cosine similarity? **A:** direction similarity of two vectors; 1=same, <0.5="moved far".
- **Q:** Adversarial perturbation? **A:** small purposeful noise (found via gradients) making AI misread.
- **Q:** PGD? **A:** iterative gradient + projecting noise to [−ε,ε] to craft adversarial noise.
- **Q:** What is EoT for? **A:** optimize noise over many transforms → robust to compression/resize.
- **Q:** STE? **A:** lets gradients flow through rounding (`x+(q-x).detach()`).
- **Q:** PSNR vs SSIM? **A:** pixel difference (dB) vs human structural perception (0..1).
- **Q:** Transferability? **A:** noise carrying to other models (usually POOR).
- **Q:** Token / Tokenizer / BPE? **A:** text chunk / text↔token converter / merge frequent char pairs.
- **Q:** Encoding (tiktoken)? **A:** per-model vocabulary (o200k_base, cl100k_base).
- **Q:** Token vs Credit? **A:** unit the model reads (countable) vs abstract billing unit.
- **Q:** TTS vs voice cloning? **A:** text→speech / read in a voice learned from a sample.
- **Q:** Zero-shot cloning? **A:** clone from a few seconds of audio, no retraining.
- **Q:** Conditioning latents? **A:** speaker embedding + context from the sample to "mimic" the voice.
- **Q:** Why WAV not webm for TTS datasets? **A:** the pipeline needs raw uncompressed PCM.
- **Q:** Fine-tune? **A:** further-train an existing model on new data (your voice).

## CS fundamentals
- **Q:** Big-O of sorting? **A:** O(n log n).
- **Q:** Hash map membership? **A:** O(1) average.
- **Q:** Process vs thread? **A:** separate memory vs shared memory.
- **Q:** SOLID — S? **A:** Single Responsibility (one class, one job).
- **Q:** DRY? **A:** Don't Repeat Yourself (e.g. a shared runner for CLI + web).
- **Q:** Git: merge vs rebase? **A:** merge keeps history / rebase rewrites it linearly.

## Testing
- **Q:** Unit vs E2E test? **A:** one unit/function vs a whole user flow.
- **Q:** What's mocking for? **A:** replace real services (LLM/fetch) for fast, stable, offline tests.
- **Q:** Property-based testing? **A:** declare an invariant, the library generates random inputs to break it.
- **Q:** CI/CD? **A:** auto-run tests on push; deploy only when passing.

## Project tie-ins (self-quiz)
- **Q:** Why is agent-talk stateless? **A:** state sent by the client each turn → easy to scale.
- **Q:** Why does quan-quen-map use two storage tiers? **A:** heavy images → IndexedDB, text → localStorage.
- **Q:** Why does word-checker skip math formulas? **A:** to reduce false positives.
- **Q:** Why does art-shield measure with cosine, not "label flip"? **A:** the label can stay the same while the embedding moved far.
- **Q:** Why does token-counter report "credits" instead of tokens for Kiro? **A:** Kiro doesn't log tokens → honest, no fabrication.
- **Q:** Why does voice-clone-web encode WAV on the client? **A:** so the backend just saves files, no heavy audio library.

> 💡 Review flashcards via **active recall**: answer before revealing — you'll retain far more than re-reading.
