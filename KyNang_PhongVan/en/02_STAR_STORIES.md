# 2. Telling stories with STAR (behavioral questions)

Behavioral questions ("Tell me about a hard bug", "A time you self-taught a new tech") should follow **STAR**:

- **S (Situation)**: brief context.
- **T (Task)**: your task/problem.
- **A (Action)**: what YOU did (the longest part — specific, technical).
- **R (Result)**: the outcome (with numbers if possible) + the lesson.

> Tip: spend ~70% on **A**. The interviewer wants to hear what *you* did, not what "the team did".

---

## 8 ready-made stories (from the projects)

### 1. "Hardest technical challenge" — Effective font (word-checker)
- **S**: I built a Word-document formatting checker.
- **T**: It flagged wrong fonts, but most paragraphs were false-positives or missed.
- **A**: I found that runs in a .docx often **don't declare a font directly** — Word resolves it across 4 inheritance tiers (run → character style → paragraph style via based_on → docDefaults). I wrote an "effective font" resolver covering all 4, reading the raw docDefaults XML via the namespace.
- **R**: No more mass false-positives; I deeply understood OOXML. Lesson: understand the data model correctly before writing checks.

### 2. "Optimization/constraint handling" — Two-tier storage (quan-quen-map)
- **S/T**: The map app stores place photos; localStorage is ~5MB and base64 images fill it instantly.
- **A**: I moved images to **IndexedDB** (large, async); localStorage keeps only `{id, category}`; on startup I load images into a cache, "hydrate" them back, and prune orphans.
- **R**: No more full localStorage; a test verifies the architecture. Lesson: pick the right store for the right data type.

### 3. "Quality assurance" — Testing crypto code (totp-manager)
- **S/T**: I implemented TOTP myself — one wrong bit makes the code useless, hard to trust "it runs = it's right".
- **A**: I verified it against the **official RFC 6238 test vectors** (SHA1/256/512) and used **property-based testing (hypothesis)** for invariants (e.g. seconds-remaining always within [1, period]).
- **R**: Confident the implementation is standard-correct. Lesson: for algorithms, official standards + fuzzing beat a few manual cases.

### 4. "Reducing noise / thinking about the user" — False-positive reduction (word-checker)
- **A**: I added many deliberate exceptions: skip math-formula paragraphs, "3,14" isn't flagged for a missing space, the reduplication "xa xa" isn't flagged as a repeated word; and I wrote **dedicated tests for "must-not-flag" cases**.
- **R**: A trustworthy tool users don't disable. Lesson: an over-flagging tool gets turned off — "better to miss than over-flag".

### 5. "Learning a new tech fast" — OAuth/Graph/IMAP (outlook-mail-reader)
- **A**: I had to understand MSAL's device-code flow, rotating refresh tokens, and falling back to IMAP XOAUTH2 when Graph lacked scopes; plus a multi-threaded, cancelable import job for many accounts.
- **R**: A stable multi-account mail reader. Lesson: break the problem down (auth → read → display) and always have a fallback.

### 6. "Designing UX for an emergency" — Panic Mode (cap-cuu-101)
- **A**: The first-aid app reads steps aloud (Web Speech); I handled async getVoices, warned when the Vietnamese voice is missing, and used a "one one five" trick so it reads "115" correctly.
- **R**: Usable hands-free in a panic. Lesson: design for the user's real context, not just the ideal case.

### 7. "Improving based on measurement" — Robust EoT noise (art-shield)
- **S**: I built a tool that adds adversarial noise to protect art from being scraped to train AI.
- **T**: Naive PGD noise looked like it "worked" but was wiped out by simple compression/resize — surviving **0/9** wash-off transforms.
- **A**: I found the root cause: the noise lives in high frequencies that smoothing filters erase. I applied **EoT (Expectation over Transformation)**: each optimization step applies several random differentiable transforms (blur/resize/compress) and averages the gradient — plus an **STE** trick to let gradients flow through the rounding (compression). I added **perceptual masking** to hide noise in detailed regions.
- **R**: Robustness jumped from **0/9 to 9/9**, with SSIM 0.94 (noise nearly invisible). Lesson: measure with the **right metric** (cosine to the original, not "label flip") and improve based on numbers, not intuition.

### 8. "Honesty with data / handling messy data" — AI log scanning (token-counter)
- **S/T**: I built an app measuring AI token/credit usage by reading local logs from Claude Code/Kiro/Antigravity — each tool logs different JSON shapes and key names, files are large and possibly corrupt.
- **A**: I wrote a **defensive** scanner: regex-filter lines before parsing, skip files >80MB, read with `errors='ignore'`, dedup via fingerprint; and detect usage objects **recursively** across schemas (`input_tokens`/`promptTokenCount`/`candidatesTokenCount`...). Most importantly: when Kiro only exposes **credits** and no tokens, I report the credits with a note and **don't fabricate** tokens.
- **R**: The app runs reliably on messy real logs with trustworthy numbers. Lesson: be defensive with real data, and **better to report "no data" than a wrong number**.

---

## How to use
- When asked a behavioral question, pick the best-fitting story, tell it S-T-**A**-R.
- One story can be reused for several questions (hard bug / proudest / learned fast).
- Always end with the **lesson** — it shows you reflect.

> 💡 Memorizing your 3 strongest stories (1, 3, 4) covers most behavioral questions. For AI/ML-leaning interviews, prioritize story **7** (improving via measurement) and **8** (data honesty).
