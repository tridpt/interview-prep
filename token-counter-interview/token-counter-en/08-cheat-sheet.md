# 08 — Cheat Sheet (last-minute skim)

## 30-second pitch
> AI Usage Monitor is a local web app to see how many tokens/credits I've spent on AI tools. The FastAPI backend has two parts: a safe local log scanner (regex filter, 80MB limit, recursive usage detection across schemas, dedup) and a token counter via tiktoken using each model's encoding. The throughline: honesty — credits-only means report credits, never fabricate tokens.

## Two pillars
1. **Token counting**: `tiktoken` + model→encoding mapping + `lru_cache`.
2. **Usage scanning**: read Claude/Kiro/Antigravity logs, defensive + honest.

## Endpoints
| Method | Path | Role |
|--------|------|------|
| GET | `/api/health` | server + tiktoken present? |
| GET | `/api/models` | preset model list |
| POST | `/api/count` | count input/output tokens |
| GET | `/api/usage/scan` | scan all logs |

## Encodings
- `o200k_base`: GPT-4o, 4.1, o3, o4-mini.
- `cl100k_base`: GPT-3.5 Turbo, older GPT-4.
- Unknown model → `o200k_base`.

## One scanner's flow (memorize)
find existing dirs → walk files by pattern → **regex-filter lines** → parse JSON (fault-tolerant) → **recursive usage detection** → **fingerprint dedup** → build `UsageRecord` → set status (`ok`/`no_usage`/`missing`).

## Safety constants
- `MAX_RECORDS = 2000`
- `MAX_SCAN_FILE_BYTES = 80MB`

## Token types aggregated
input, output, cache_read, cache_creation, reasoning, total + credits + context_percent.

## One-line glossary
- **Token**: text chunk, billing unit.
- **BPE**: merge frequent char pairs into tokens.
- **Encoding**: tokenizer vocabulary.
- **Credit**: abstract billing unit (≠ token).
- **lru_cache**: remember the encoder.
- **Debounce**: defer call until typing stops (220ms).
- **Fingerprint**: record id string for dedup.

## Point-scorers
- **Defensive** scanning: regex filter, size limit, errors="ignore", catch OSError.
- **Recursive** usage detection across schemas (input_tokens / promptTokenCount / candidatesTokenCount...).
- **Honesty**: Kiro has only credits → note it, don't fake tokens.
- **Privacy**: local-only, read-only, no networking.
- **escapeHtml** XSS protection on the frontend.

## Go-to line on limits
> "The app only gets what a tool writes locally. Many tools expose only credits or write nothing, so I report exactly what's there with a clear note — better incomplete than wrong. tiktoken is also only exact for OpenAI models; for Claude/Gemini it's an estimate."

## Common pitfalls
- StaticFiles `/` mount goes **last** (after /api).
- Wrong encoding → wrong token count.
- JSON parsing must be fault-tolerant (log prefix before `{`).
- Adding a usage key means editing **both** backend and frontend.
