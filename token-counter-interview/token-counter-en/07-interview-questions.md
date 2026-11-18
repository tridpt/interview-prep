# 07 — Interview Questions (with model answers)

---

## A. Overview (1–10)

**1. What does the project do in one sentence?**
A local web app that reads AI tools' logs to aggregate real usage when metadata exists, and estimates tokens for text via tiktoken.

**2. The two main modes?**
(1) Scan local logs for real usage; (2) manual token estimate via tiktoken for pasted text.

**3. Who uses it?**
Users of AI coding tools wanting to track how many tokens/credits they've spent.

**4. Main technologies?**
FastAPI + uvicorn, tiktoken, Pydantic, Python stdlib for scanning; vanilla JS frontend.

**5. Why does counting tokens matter?**
Vendors bill by tokens; counting ahead helps budget cost.

**6. Supported log sources?**
Claude Code (tokens if JSONL/telemetry), Kiro (credits + context %), Google Antigravity (tokens if metadata in logs), and manual API-JSON paste.

**7. Main endpoints?**
`/api/health`, `/api/models`, `/api/count` (POST), `/api/usage/scan`.

**8. Does data leave the machine?**
No. Scanning is read-only, fully local, no networking.

**9. The app's honesty point?**
It won't fabricate tokens when a vendor only exposes credits; it states tiktoken is an estimate for non-OpenAI models.

**10. Do the two areas depend on each other?**
No — missing tiktoken still leaves scanning working, and vice versa.

---

## B. Foundational concepts (11–26)

**11. What is a token?**
A text chunk — the unit the model reads and is billed by (could be a word, sub-word, punctuation).

**12. What is a tokenizer?**
Converts text ⇄ a list of token ids.

**13. What is BPE?**
Byte Pair Encoding: progressively merge frequent character/byte pairs into larger tokens until the vocab is full.

**14. What is an encoding in tiktoken?**
A vocabulary + merge rules tied to a model family: `o200k_base` (GPT-4o/4.1/o3...), `cl100k_base` (older GPT-3.5/4).

**15. Why map model→encoding?**
Counting with the wrong encoding gives a wrong number; each model has its own encoding.

**16. Does tiktoken count exactly for all models?**
Exact for OpenAI models; for Claude/Gemini it's an approximation.

**17. Input vs output tokens?**
Input = prompt sent; output = text the model generates (usually pricier).

**18. How do credits differ from tokens?**
Credits are an abstract billing unit of some tools (like Kiro), not a token count.

**19. What are cache and reasoning tokens?**
Cache tokens = read from cache (cheaper); reasoning tokens = "thinking" of o-series models. The app aggregates both.

**20. What is FastAPI?**
An async Python web framework, validates input via Pydantic, auto-generates docs.

**21. What's Pydantic used for here?**
Defining `CountRequest` so FastAPI auto-parses + validates the POST body.

**22. What does lru_cache do?**
Remembers `_encoder_for(model)` results to avoid recreating the encoder on each keystroke.

**23. Where are dataclasses used?**
`UsageRecord`, `SourceScan` in the scanner — concise, easy `asdict()`.

**24. Why filter lines by regex before parsing JSON?**
JSON parsing is costly; quick filtering skips most irrelevant lines in big log files.

**25. What is recursion used for?**
Walking nested JSON (unknown depth) to find usage objects.

**26. What is debounce?**
Deferring a call until the user stops typing (220ms) to avoid API spam.

---

## C. Backend detail (27–44)

**27. Why import tiktoken with try/except?**
So the app still boots without tiktoken; `/api/health` reports the status.

**28. Why import scan_all two ways?**
To run whether uvicorn is invoked from root (`backend.main:app`) or inside backend/ (`main:app`).

**29. How many cases does `_encoder_for` handle?**
3: direct encoding name; `encoding_for_model`; fallback by table → ultimately `o200k_base`.

**30. How does `_count` handle None text?**
`encode(text or "")` — treats None/empty as an empty string.

**31. What does `/api/count` return?**
input_tokens, output_tokens, total_tokens, input_chars, output_chars, model, encoding.

**32. What are `MAX_RECORDS` and `MAX_SCAN_FILE_BYTES` for?**
Cap at 2000 records and skip files >80MB to avoid hanging.

**33. How does `scan_all` merge data?**
Calls 3 scanners, `pop("_records")` to merge, sorts newest first, trims 2000, computes totals.

**34. Describe a scanner's steps.**
Find existing dirs → walk files by pattern → regex-filter lines → parse JSON → detect usage → fingerprint dedup → build UsageRecord → set status.

**35. What status values can a source have?**
`missing` (no dir), `no_usage` (dir but no data), `ok` (records).

**36. What's special in `scan_kiro`?**
Reads credits from `meteringEvent` and context %, records `kind="credit_usage"` with a clear "no token split" note.

**37. How does `_looks_like_usage` decide?**
True if the dict has any key in the usage-keys set, or a nested `usageMetadata` is usage too.

**38. How does `_record_from_usage` normalize?**
Uses `_int_first` trying many key names per token type; total = taken directly or summed.

**39. How does `_parse_json_line` tolerate errors?**
Tries the whole line, then from the first `{` (strip log prefix); returns None on failure.

**40. What is the dedup fingerprint?**
A string of file + request_id/timestamp + usage content; skip if already seen.

**41. What does `_clean_model` do?**
Trims a model id repeated twice in Kiro logs (`claude-...claude-...`).

**42. Where does `_timestamp_from_path` get the time?**
Extracts a `\d{8}T\d{6}` pattern from the filename; else uses mtime.

**43. Why `errors="ignore"` in `_iter_matching_lines`?**
So undecodable characters don't crash file reading.

**44. What does `_totals` compute?**
Sum of all token types + credits (6-digit rounding) for global and per-source totals.

---

## D. Frontend & deep questions (45–60)

**45. Why `escapeHtml`?**
XSS protection — log data is untrusted, escape HTML before rendering.

**46. How is debounce implemented?**
`clearTimeout` then `setTimeout(fn, 220)` on each keystroke.

**47. Why render only 250 record rows?**
Keep the DOM light despite thousands of records.

**48. Where does the usage-parsing logic live?**
Both backend (Python) and frontend (JS) — same recursive + many-keys philosophy.

**49. The trade-off of duplicating parse logic?**
Convenient for the client (instant feedback) but must sync both places when adding new keys.

**50. What about a 5GB log file?**
Skipped, exceeding 80MB — a safety/completeness trade-off; could raise the limit or stream if needed.

**51. What if a tool's log schema changes?**
Just add the new key to the usage-keys set; the key-driven design is easy to extend.

**52. Privacy risk?**
Reading logs may touch sensitive data; the app minimizes via read-only, metadata-only, no networking.

**53. Why not persist scan results?**
Stateless design: each scan re-reads the source logs → always fresh, no data copies.

**54. How to add support for a new AI tool?**
Write another `scan_xxx()` in the same shape (find dirs → filter → parse → detect usage), add to `scan_all`.

**55. Why count tokens on the backend, not the client?**
tiktoken is a Python library; accurate counting needs server-side. (The client only parses existing usage JSON.)

**56. Biggest limitation?**
It depends on whether a tool writes metadata locally; many only expose credits or write nothing.

**57. Next improvements?**
Time-series charts, cost estimates (tokens × unit price), more sources, CSV export.

**58. Why include generic encodings (`o200k_base`) in the model list?**
To let users count by a specific table directly, bypassing a model name.

**59. What if the pasted JSON has no usage?**
`findUsage` returns null → shows "No usage object found", no crash.

**60. Biggest lesson from the project?**
Handling real data demands defensiveness and honesty: better to report "no data" than a wrong number.

---

## Answering tips
- Emphasize the two pillars: **accurate token counting** (tiktoken+cache) and **defensive + honest log scanning**.
- When discussing logs, give a concrete example (Kiro has only credits → clear note).
- Show security thinking: local-only, read-only, escapeHtml.
