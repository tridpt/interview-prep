# 📑 Index — Token Counter (AI Usage Monitor) Interview Prep

Welcome! This set helps you understand the **AI Usage Monitor** project (folder: `token-counter`) for interviews, even if you know nothing about tokens or tokenizers. Read `00 → 08` in order.

> It's a **local web app** with two modes: (1) **scan local logs** of AI coding tools (Claude Code, Kiro, Google Antigravity) to read actual usage when metadata exists; (2) **estimate tokens** for pasted text using `tiktoken`.

## Reading order

| File | Title | Content | When |
|------|-------|---------|------|
| `00` | **Index** (this file) | Map of docs | First |
| `01` | [Project overview](01-project-overview.md) | What it is, the problem, main tech | Big picture |
| `02` | [Foundational concepts](02-foundational-concepts.md) | Token, tokenizer, BPE, encoding, FastAPI, async — from zero | Theory base |
| `03` | [System architecture](03-system-architecture.md) | Backend/frontend, data flow, APIs | After concepts |
| `04` | [Backend deep dive](04-backend-deep-dive.md) | `main.py` (counting) + `usage_scanner.py` (scanning) function by function | The core |
| `05` | [Frontend deep dive](05-frontend-deep-dive.md) | `app.js`, `index.html`, JSON usage parsing, debounce | The UI |
| `06` | [Standout features](06-standout-features.md) | Point-scorers: safe scanning, recursive usage detection, data honesty | Pick highlights |
| `07` | [Interview questions](07-interview-questions.md) | 60+ Q&A | Practice |
| `08` | [Cheat sheet](08-cheat-sheet.md) | One-page recap | Last minute |

## After reading you should be able to

1. Pitch the project in 2 minutes: what, for whom, with what tech.
2. Explain token/tokenizer/BPE and why counting tokens matters (API cost).
3. Understand **safe log scanning**: regex pre-filter, file-size limit, dedup via fingerprint, recursive usage detection across many key names.
4. Articulate the **data honesty** philosophy: the app won't fabricate tokens when a vendor only exposes credits/quota.

## One line to remember

> "AI Usage Monitor is a local web app that reads AI tools' logs to aggregate real usage when metadata exists, and estimates tokens for text via `tiktoken`. Its core is **safe, read-only, and honest** — it won't pretend to have exact tokens when a vendor only exposes credits."
