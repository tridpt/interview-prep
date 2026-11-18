# 03 — System Architecture

## 1. Big picture

```
┌─────────────────────────────┐         ┌────────────────────────────┐
│   FRONTEND (HTML/CSS/JS)     │         │   BACKEND (FastAPI)        │
│                             │  HTTP   │                            │
│  - Totals metric cards      │ ◄─────► │  main.py                   │
│  - Sources list             │  /api   │   /api/health              │
│  - Recent records table     │         │   /api/models              │
│  - Manual token counter     │         │   /api/count   ──► tiktoken│
│  - Paste usage JSON box      │         │   /api/usage/scan ─┐       │
└─────────────────────────────┘         │                    ▼       │
                                         │     usage_scanner.py       │
                                         │   scan_claude_code()       │
                                         │   scan_kiro()              │
                                         │   scan_antigravity()       │
                                         └──────────┬─────────────────┘
                                                    ▼
                                   Local logs on disk (read-only):
                                   ~/.claude, %APPDATA%/Kiro/logs, ...
```

## 2. Two independent feature areas

The project bundles **two tools** into one app, sharing the FastAPI shell + frontend:

| Area | API | Module | Depends on |
|------|-----|--------|-----------|
| **Token counting/estimate** | `/api/count`, `/api/models` | `main.py` | `tiktoken` |
| **Real usage scanning** | `/api/usage/scan` | `usage_scanner.py` | stdlib |

The two are **independent**: if `tiktoken` is missing, counting reports a clear error but scanning still works, and vice versa.

## 3. Endpoints

| Method | Path | Role | Returns |
|--------|------|------|---------|
| GET | `/api/health` | Server + is tiktoken present | `{status, tokenizer}` |
| GET | `/api/models` | Preset model list | `{models: [...]}` |
| POST | `/api/count` | Count input/output tokens by model | `{input_tokens, output_tokens, total, encoding...}` |
| GET | `/api/usage/scan` | Scan all local logs | `{totals, sources, records}` |
| (mount) | `/` | Serve static frontend | index.html |

## 4. Data flow — Token counting

```
1. User types in Input/Output, or picks a model
2. frontend debounces 220ms → POST /api/count {model, input_text, output_text}
3. main.py:
   - Pydantic parses into CountRequest
   - _encoder_for(model): look up ENCODING_BY_MODEL, get encoder (lru cached)
   - _count(text): encoder.encode(text) then len()
4. return {input_tokens, output_tokens, total_tokens, input_chars, output_chars, encoding}
5. frontend updates the numbers
```

## 5. Data flow — Usage scanning

```
1. User clicks "Scan Local Logs" (or auto on page load)
2. GET /api/usage/scan
3. scan_all() calls in turn:
     scan_claude_code(), scan_kiro(), scan_antigravity()
   each:
     - find existing log dirs (_existing_paths)
     - walk files by pattern (_iter_files)
     - filter lines by regex (_iter_matching_lines)  ← optimization
     - parse JSON per line (_parse_json_line)
     - detect usage objects (recursive) → build UsageRecord
     - dedup via fingerprint/seen set
4. scan_all() merges records, sorts by time, trims MAX_RECORDS (2000), computes totals
5. return JSON → frontend renders sources + records table + 4 totals
```

## 6. Data model (dataclasses)

```python
@dataclass
class UsageRecord:        # one usage record
    source, kind, timestamp, model, path, request_id
    input_tokens, output_tokens, cache_read_tokens,
    cache_creation_tokens, reasoning_tokens, total_tokens
    credits, context_percent, note

@dataclass
class SourceScan:         # result of scanning one source
    id, name, status, paths, files_scanned, records, note, totals
```
- A source's `status` has three values: `missing` (dir not found), `no_usage` (dir present but no data), `ok` (records found).
- `kind`: `token_usage` (has tokens) or `credit_usage` (credits only, like Kiro).

## 7. How the backend embeds the frontend

```python
BASE_DIR = Path(__file__).resolve().parent.parent
FRONTEND_DIR = BASE_DIR / "frontend"
app.mount("/", StaticFiles(directory=FRONTEND_DIR, html=True))
```
Mount `/` at the **end of the file** (after all `/api` routes) so it doesn't hide the APIs.

## 8. "Soft" imports for resilience

```python
try:
    from usage_scanner import scan_all       # run from backend/
except ModuleNotFoundError:
    from backend.usage_scanner import scan_all  # run from project root

try:
    import tiktoken
except ImportError:
    tiktoken = None                          # still boots, reported via /api/health
```
- Two import styles so it runs no matter which dir `uvicorn` is invoked from.
- `tiktoken` soft import: if missing, `tiktoken = None`, the app still boots, and counting returns a 500 with install instructions.

## 9. Core design principles

1. **Read-only, local-only**: the scanner never writes/edits/deletes logs, never networks.
2. **Defensive against real files**: real logs are messy, huge, possibly corrupt → every step has try/except and limits.
3. **Data honesty**: clearly distinguish tokens (exact) vs credits (abstract) vs estimates (tiktoken for non-OpenAI models).
4. **Two separate tools**: a failure in one area doesn't bring down the other.
