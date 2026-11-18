# 01 — Project Overview

## 1. What is this project? (for a complete beginner)

When you use AI coding tools (Claude Code, Kiro, Google Antigravity, or call the OpenAI API...), every conversation consumes **tokens** — the billing unit of language models. More tokens = more money. But tools often **don't clearly show** how much you've used.

**AI Usage Monitor** is a **web app running right on your machine** that answers "how many tokens/credits have I used?". It has two modes:

1. **Local log scanner (Actual usage)**: finds and reads the log files those AI tools write to your disk, extracting token/credit metadata *when present*.
2. **Manual tokenizer estimate**: you paste text (prompt + answer) and the app counts exact tokens per model using `tiktoken`.

There's also a **paste API usage JSON** box to read the real usage numbers a provider returned.

## 2. What problem does it solve?

| Problem | How it's handled |
|---------|------------------|
| You don't know how many tokens/credits you've spent | Scan AI tools' local logs and aggregate |
| You want to estimate tokens BEFORE calling an API (to budget) | Count with `tiktoken` using the model's encoding |
| A vendor only exposes credits, not tokens | The app is **honest**: reports "no tokens, only credits" instead of faking |
| Privacy concerns | Scanning is **read-only, fully local**, nothing sent anywhere |

## 3. Main technologies

| Component | Tech | Role |
|-----------|------|------|
| Backend | **FastAPI** + `uvicorn` | Token-count API + log scan |
| Token counting | **tiktoken** (OpenAI) | Accurate BPE tokenizer per model |
| Input validation | **Pydantic** (`BaseModel`) | Validate request body |
| Log scanning | Python stdlib (`pathlib`, `re`, `json`, `dataclasses`) | Find/read/parse log files |
| Frontend | Vanilla HTML/CSS/JS | Usage tables, token counter, JSON parse |

## 4. Directory structure

```
token-counter/
├── backend/
│   ├── main.py            # FastAPI: /api/count, /api/models, /api/usage/scan, /api/health
│   └── usage_scanner.py   # Scan Claude Code / Kiro / Antigravity logs
├── frontend/
│   ├── index.html         # UI: metrics, sources, records table, token counter
│   ├── app.js             # Call APIs, render, parse usage JSON, debounce
│   └── style.css
├── requirements.txt       # fastapi, uvicorn, tiktoken
├── start_token_counter.bat
└── start_usage_monitor.bat
```

## 5. Supported log sources

| Source | What can be read locally |
|--------|--------------------------|
| **Claude Code** | Real token usage if JSONL/telemetry contains `usage` metadata |
| **Kiro** | Credits used + context % from local logs (NO input/output token split) |
| **Google Antigravity** | Real tokens only if logs contain metadata; otherwise reports "none found" |
| **API JSON** (manual paste) | Real usage if the response has a `usage` object |

> Key philosophy: if a vendor only exposes quota/credits and doesn't write input/output tokens locally, **the app won't pretend to have exact tokens**. This honesty is emphasized in the README.

## 6. What makes it notable

- **Defensive log scanning**: regex pre-filter before parsing, file-size limit (80MB), skip broken files, dedup via fingerprint.
- **Recursive `usage` detection**: each tool's logs have different JSON shapes, so the app walks the JSON tree to find any object "that looks like usage" across many key names (`input_tokens`, `prompt_tokens`, `promptTokenCount`...).
- **Fully local, read-only**: privacy-respecting — nothing leaves the machine.
- **Correct-encoding counting**: model → encoding mapping (`o200k_base`, `cl100k_base`), encoder cached with `lru_cache`.

## 7. 30-second pitch

> "AI Usage Monitor is a local web app that tells me how many tokens/credits I've spent on AI tools. The FastAPI backend has two parts: a safe local log scanner — regex pre-filter, size limit, recursive usage detection across schemas — and a token counter using tiktoken with each model's encoding. The throughline is honesty: if a vendor only exposes credits, I report credits, I don't fabricate tokens."
