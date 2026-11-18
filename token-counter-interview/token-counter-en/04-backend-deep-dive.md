# 04 — Backend Deep Dive (function by function)

The backend has two files: `main.py` (token counting + API shell) and `usage_scanner.py` (log scanning). This is the "guts".

---

# PART 1 — `main.py`

## A. Setup and soft imports
```python
try:    from usage_scanner import scan_all
except ModuleNotFoundError: from backend.usage_scanner import scan_all
try:    import tiktoken
except ImportError: tiktoken = None
```
- Two import styles to run regardless of where `uvicorn` is invoked (`backend.main:app` from root, or `main:app` inside backend/).
- Soft `tiktoken`: missing doesn't crash the app.

## B. Model list and encoding mapping
```python
MODEL_PRESETS = [
    {"id": "gpt-4.1",  "label": "GPT-4.1", "encoding": "o200k_base"},
    {"id": "gpt-4o",   "label": "GPT-4o",  "encoding": "o200k_base"},
    {"id": "gpt-3.5-turbo", ..., "encoding": "cl100k_base"},
    {"id": "o200k_base", ...}, {"id": "cl100k_base", ...},  # generic encodings
]
ENCODING_BY_MODEL = {item["id"]: item["encoding"] for item in MODEL_PRESETS}
```
- Each model maps to its encoding; two "generic" entries let you pick an encoding directly.

## C. `_require_tiktoken()`
If `tiktoken is None` → raise `HTTPException(500, "Missing tiktoken. Run: pip install...")`. → An actionable error instead of a confusing stack trace.

## D. `_encoder_for(model)` — get the encoder (cached)
```python
@lru_cache(maxsize=32)
def _encoder_for(model):
    tk = _require_tiktoken()
    if model in {"o200k_base","cl100k_base","p50k_base","r50k_base"}:
        return tk.get_encoding(model)          # pick encoding directly
    try:
        return tk.encoding_for_model(model)    # let tiktoken infer
    except KeyError:
        return tk.get_encoding(ENCODING_BY_MODEL.get(model, "o200k_base"))  # fallback
```
Three tiers: (1) if it's an encoding name, use it; (2) try tiktoken's model→encoding mapping; (3) unknown model → fallback by table, ultimately `o200k_base`. `lru_cache` remembers the encoder to avoid recreating it.

## E. `_count(text, model)`
`return len(_encoder_for(model).encode(text or ""))` — encode then count. `text or ""` safely handles `None`/empty.

## F. The endpoints
- `GET /api/health` → `{status:"ok", tokenizer: tiktoken is not None}`.
- `GET /api/models` → returns `MODEL_PRESETS`.
- `POST /api/count`: takes a `CountRequest`, counts input + output, also returns `encoding`, char counts, total. The type-as-you-go counter API.
- `GET /api/usage/scan` → directly calls `scan_all()`.

---

# PART 2 — `usage_scanner.py` (the hardest, most valuable part)

## G. Safety constants
```python
MAX_RECORDS = 2000                      # cap records returned
MAX_SCAN_FILE_BYTES = 80 * 1024 * 1024  # skip files > 80MB
```
Avoid hanging on huge logs or too many records.

## H. `scan_all()` — orchestration
```python
sources = [scan_claude_code(), scan_kiro(), scan_antigravity()]
all_records = []
for source in sources:
    all_records.extend(source.pop("_records", []))   # gather records, drop from payload
all_records.sort(key=lambda r: r.timestamp or "", reverse=True)
all_records = all_records[:MAX_RECORDS]
totals = _totals(all_records); totals["files_scanned"] = sum(...)
return {scanned_at, totals, sources, records: [asdict(r) ...]}
```
- Each scan returns a dict with a temp `_records` key; `scan_all` `pop`s it to merge, then removes it from the source payload.
- Sorts newest first, trims 2000, computes totals.

## I. `scan_claude_code()` — the model scanner
Steps (worth memorizing, they repeat in all three scanners):
1. **Find existing dirs**: `~/.claude/projects`, `%APPDATA%/Claude/local-agent-mode-sessions`.
2. **Walk files** `*.jsonl`, `*.log`.
3. **Filter lines by regex** containing `usage|input_tokens|output_tokens|...` (optimization — skip irrelevant lines).
4. **Parse JSON** per line.
5. **Find the usage object**: prefer specific paths `message.usage` / `usage`, otherwise recursive search.
6. **Extract model, request_id, timestamp** from several possible locations.
7. **Fingerprint dedup**: `f"{file}:{request_id or timestamp}:{json.dumps(usage)}"` — don't count the same record twice.
8. Build a `UsageRecord` via `_record_from_usage(...)`.
Finally set `status` = `ok`/`no_usage`/`missing` and attach `_records`.

## J. `scan_kiro()` — handling "credits, not tokens"
- Find `%APPDATA%/Kiro/logs`, `~/.kiro`; walk files named `5-Q Chat API*.log`.
- Filter lines with `meteringEvent|usageSummaryEntry|contextUsageEvent`.
- Walk `response.events[]`: accumulate `meteringEvent.usage` into **credits**, read `contextUsagePercentage`, detect `modelId`.
- If no credits in events, follow a deeply nested path `fullMessage.message.kwargs.additional_kwargs.usageSummaryEntry`.
- Skip if both credits and context are empty.
- Build `UsageRecord(kind="credit_usage", credits=..., context_percent=..., note="Kiro log reports credits, not exact token split.")`.
→ A concrete demonstration of the **honesty** philosophy: Kiro exposes no tokens, so the app records credits with a clear note.

## K. `scan_antigravity()`
- Find `%APPDATA%/Antigravity`, `~/.antigravity`; walk `logs/*.log|*.jsonl` + a few special files (`app_storage.json`, `Local State`).
- A broad pattern (including Google-style keys `promptTokenCount`, `candidatesTokenCount`, `totalTokenCount`).
- For each JSON line, **find ALL** usage objects (`_find_all_usage_objects`) → each becomes a record (deduped by fingerprint).

## L. Key utility functions

### `_iter_matching_lines(file, pattern)` — safe scanning
```python
if file.stat().st_size > MAX_SCAN_FILE_BYTES: return   # skip huge files
regex = re.compile(pattern, re.IGNORECASE)
with file.open("r", encoding="utf-8", errors="ignore") as f:
    for line in f:
        if regex.search(line): yield line
except (OSError, UnicodeError): return                  # broken file → skip quietly
```
Three defensive layers: size limit, `errors="ignore"` for bad chars, catch OSError/UnicodeError.

### `_parse_json_line(line)`
Tries to parse the whole line; if it fails, tries from the first `{` (logs often prefix timestamp/level before the JSON). Returns `None` on failure → doesn't crash the loop.

### `_find_all_usage_objects(obj)` + `_looks_like_usage(value)` — the scanner's heart
```python
def walk(value):
    if isinstance(value, dict):
        if _looks_like_usage(value): found.append(value)
        for child in value.values(): walk(child)
    elif isinstance(value, list):
        for child in value: walk(child)
```
- **Recursively** walks every corner of the JSON.
- `_looks_like_usage`: True if the dict has **any** key in the usage-keys set (`input_tokens`, `prompt_tokens`, `promptTokenCount`, `candidatesTokenCount`...) — covering many vendor schemas.

### `_record_from_usage(...)` — normalize to a common schema
Uses `_int_first(usage, [many_key_names])` to get tokens regardless of vendor naming:
- input: `input_tokens` / `prompt_tokens` / `inputTokenCount` / `promptTokenCount`
- output: `output_tokens` / `completion_tokens` / `candidatesTokenCount`...
- cache, reasoning similarly.
- total: take `total_tokens` if present, else sum the components.

### `_int_first`, `_first_text`, `_get_path`, `_optional_float`, `_float`
A "safe getter" toolkit: try multiple keys, cast carefully (return `None`/0 on error), follow nested paths without crashing when a key is missing.

### `_timestamp_from_path(path)`
Tries to extract a timestamp from the filename (pattern `\d{8}T\d{6}`); otherwise uses the file's mtime.

### `_clean_model(model)`
Some Kiro logs repeat the model id twice (`claude-...claude-...`) → this function trims it to one. A "clean messy real data" detail.

### `_totals(records)`
Sums all token types + credits (rounded to 6 digits) → used for both the global total and per-source totals.

---

## Backend spirit, summarized
> The counting part is **accurate and simple** (tiktoken + cache). The scanning part is **defensive and flexible**: fast regex filtering, fault-tolerant parsing, recursive usage detection across schemas, normalization into one common record, and honest separation of tokens vs credits.
