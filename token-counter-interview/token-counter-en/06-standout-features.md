# 06 — Standout Features (your point-scorers)

---

## 1. Defensive scanning against messy real data ⭐⭐⭐

**Why it stands out**: real logs are huge, messy, possibly corrupt. The scanner handles every case without crashing.

**How to say it**: "Before expensive JSON parsing, I filter lines by regex; skip files over 80MB; read with `errors='ignore'`; catch OSError/UnicodeError so a broken file doesn't sink the whole scan. Each record has a fingerprint to avoid double-counting."

## 2. Recursive usage detection across schemas ⭐⭐⭐

**Why it stands out**: each AI tool (Claude, Kiro, Gemini/Antigravity) logs different JSON shapes and key names. One algorithm covers all.

**How to say it**: "I walk the whole JSON tree recursively, treating a dict as 'usage' if it has any key in a known set — `input_tokens`, `prompt_tokens`, `promptTokenCount`, `candidatesTokenCount`... Then I normalize into one common record by trying several key names."

## 3. The "data honesty" philosophy ⭐⭐⭐

**Why it stands out**: rare and mature. The app doesn't fabricate numbers.

**How to say it**: "If a vendor only exposes credits and context % without tokens (like Kiro on the test machine), I report the credits with a 'no token split' note rather than inferring fake tokens. I also state clearly that tiktoken is OpenAI's tokenizer, so for Claude/Gemini it's just an estimate."

## 4. Correct-encoding counting + cache ⭐⭐

**How to say it**: "I map model → encoding (`o200k_base`, `cl100k_base`) with a 3-tier fallback, and wrap it in `lru_cache` so I don't recreate the encoder on every keystroke — important since the counter is called continuously via debounce."

## 5. Fully local, read-only (privacy-first) ⭐⭐

**How to say it**: "The scanner only reads, never writes/edits/deletes, and never networks. Everything runs on the user's machine. That's the core privacy commitment."

## 6. Two separate tools, independently fault-tolerant ⭐

**How to say it**: "Soft-import tiktoken: if missing, counting reports a clear error via `/api/health`, but the log scanner still works. The two areas don't drag each other down."

## 7. XSS protection on the frontend ⭐

**How to say it**: "Log data is untrusted, so I `escapeHtml` every value before injecting into innerHTML — preventing injection via unusual paths/models in the logs."

## 8. Usage parsing works on client and server ⭐

**How to say it**: "The 'paste usage JSON' box parses right in the browser with the same recursive logic as the backend, for instant feedback without a round-trip."

---

## "What to emphasize by JD" table

| JD emphasizes | Emphasize this |
|---------------|----------------|
| Backend / API (Python) | FastAPI, Pydantic, lru_cache, soft imports, clean endpoint design |
| Data / log processing | defensive scanning, recursive usage detection, schema normalization, fingerprint |
| AI / LLM tooling | token/tokenizer/BPE, tiktoken, input/output/cache/reasoning tokens |
| Security / privacy | local-only read-only, escapeHtml, no networking |
| Frontend | plain DOM, debounce, capped render, UI state |
