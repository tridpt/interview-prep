# 4. Backend Deep Dive (file by file)

This file dissects each backend part. Read it alongside the actual code in `agent-talk/backend/`.

---

## 4.1. `config.py` — Provider configuration

**Job:** Read API keys & model info from the `.env` file, define the "providers" (LLM vendors).

**Key points:**
```python
ROOT = Path(__file__).resolve().parent.parent
load_dotenv(ROOT / ".env")   # load environment variables from .env
```
- Uses `pathlib.Path` to compute the project root reliably (independent of the current working directory).
- `load_dotenv` reads the `.env` file into environment variables.

```python
PROVIDERS = {
    "default": _provider("LLM_", "https://api.groq.com/openai/v1", "llama-3.3-70b-versatile", "Groq"),
    "alt":     _provider("LLM2_", "", "", "Provider phụ"),
}
```
- Two providers: `default` (Groq) and `alt` (optional, e.g. Gemini).
- The `_provider(prefix, ...)` helper reads `{prefix}BASE_URL`, `{prefix}API_KEY`, ... with defaults if missing.

**Two important functions:**
- `get_provider(id)`: get a provider by id, **falling back to default** if missing → safe, never returns None.
- `public_providers()`: returns **safe** info for the frontend — includes `configured: bool(api_key)` but **never sends the real key out**. (A security point worth mentioning.)

> 💡 **Why a separate config?** To have one single place managing sensitive config; other modules just "request" a provider rather than reading env themselves. Easy to change, easy to test.

---

## 4.2. `llm.py` — The LLM-calling layer

**Job:** Encapsulate calling the OpenAI-compatible LLM API, both streaming and non-streaming, with clean error handling.

**`LLMError`**: a dedicated exception. Every error (network, status code, bad format) is "wrapped" into `LLMError` → the upper layer (`main.py`) only needs to catch this one type.

**`_prepare(...)`**: a shared helper that builds `url`, `headers` (sets `Authorization: Bearer {key}`), and `payload` (model, messages, temperature, max_tokens, stream). If the provider has no key → raise `LLMError` immediately.

**`chat(...)` (non-streaming):**
```python
async with httpx.AsyncClient(timeout=60) as client:
    resp = await client.post(url, headers=headers, json=payload)
...
return data["choices"][0]["message"]["content"].strip()
```
- Used for "one-shot" tasks: picking a speaker, summarizing, scoring, memory, thinking up a query.
- Catches `httpx.HTTPError` (network errors) and status codes != 200, turning them into `LLMError`.

**`chat_stream(...)` (streaming — async generator):**
```python
async with client.stream("POST", url, ...) as resp:
    async for line in resp.aiter_lines():
        if not line.startswith("data:"): continue
        data = line[len("data:"):].strip()
        if data == "[DONE]": break
        obj = json.loads(data)
        delta = obj["choices"][0]["delta"].get("content")
        if delta: yield delta
```
- Reads each SSE line, extracts `delta.content`, `yield`s progressively.
- `timeout=120` (higher because streaming takes longer).
- Wrapped in `try/except` so any per-line parse error is **safely skipped** (`continue`) — one bad line doesn't kill the whole stream.

> 💡 **Why both chat and chat_stream?** Two different needs: dialogue needs streaming (typing UX); internal tasks (summarize, score) need the whole result at once to process further → non-streaming is simpler.

---

## 4.3. `search.py` — Web lookup

**Job:** Let agents fetch real evidence. Prefer Tavily (if a key exists), fall back to DuckDuckGo (free, no key).

**Core philosophy (state in the interview):** *"Search is an auxiliary feature; it must not crash the main flow."* Therefore:
```python
async def web_search(query, max_results=4):
    ...
    try:
        if TAVILY_KEY:
            res = await _tavily(...)
            if res: return res
        return await _duckduckgo(...)
    except (httpx.HTTPError, Exception):
        return []   # any error → return empty, the agent speaks normally
```
It **never raises to the outside** — a deliberate design decision (graceful degradation).

**`_duckduckgo(...)`**: calls `https://html.duckduckgo.com/html/`, then **uses regex to extract** title/link/snippet from the HTML. (This is "scraping" the HTML page, since the free DuckDuckGo has no JSON API.)
- `_clean()`: strips HTML tags (`_TAG_RE`) and decodes HTML entities (`html.unescape`).

**`_tavily(...)`**: calls the proper JSON API, cleaner.

**`format_results(...)`**: formats results into text `[1] Title — snippet (source: url)` to inject into the LLM prompt.

> ⚠️ **An honest weakness**: scraping DuckDuckGo HTML with regex is **brittle** — if they change the HTML, the regex breaks. It's a trade-off to get a free, no-key feature. A more robust solution: use Tavily or an official search API.

---

## 4.4. `main.py` — The orchestration layer (the heart of the project)

### Pydantic models (define & validate requests)
- `Agent`: name, persona, provider, goal, memory.
- `TurnRequest`, `NextRequest`, `SummarizeRequest`, `ScoreRequest`, `MemoryRequest`: one clear schema per endpoint, with defaults.

### `build_messages(req)` — The prompt builder (most important function)
Returns `(me, messages)`:
1. Find the speaking agent (`_find`), list the others.
2. The "who is talking" line: group (>2 agents) or one-on-one.
3. Assemble the system prompt: personality + topic + **hidden goal** (told not to reveal it) + **memory** + **debate phase** (opening/rebuttal/closing) + reply rules (concise 2-4 sentences, full Vietnamese diacritics, no action descriptions, no playing others).
4. If there's a `summary` → add a system message "Summary of earlier events".
5. **Role mapping**: iterate `history`, `me`'s words → `assistant`, others' words → `user` as `"Name: content"`.
6. Edge cases: no history → add a primer "Start the conversation..."; last turn is assistant → add "(continue the conversation)" (the LLM needs to end on a user message to reply).

### `maybe_search_context(req, me)` — Conditional lookup
- Only runs when `allow_search=True`.
- **Two steps**: (1) ask the LLM "do you need a lookup, if so give one query" (low temperature 0.2, max_tokens 40); if the LLM replies "NO" → skip. (2) if there's a query → call `search.web_search`, format it, return a system message.
- **Never raises** (wrapped in try/except). This is the **"let the LLM decide if it needs real facts"** technique — smart and economical.

### `/api/turn_stream` — The central endpoint
- Build messages → check api_key → (optionally) insert lookup context at position 1 → `StreamingResponse` wrapping the `chat_stream` generator.
- An LLM error mid-stream → `yield "\n[ERROR] ..."` (doesn't crash the connection).
- If there are sources → attach an `X-Sources` header (JSON `quote`-encoded for character safety).

### `/api/next_speaker` — Pick the next speaker
- Send the recent flow + the last speaker, ask the LLM "who should speak next" (temperature 0.3).
- **Match the name** in the reply; if no match → **round-robin fallback** (take the next person in the list). Always returns a valid name.

### `/api/summarize` — Compress context
- Summarize in 3-5 sentences, merging with the previous summary (rolling summary). temperature 0.3.

### `/api/score` — AI judge (notable for JSON handling)
- The prompt asks for **JSON** with 4 criteria (logic, evidence, creativity, attitude) + overall + winner.
- **Defensive parsing**:
  - `re.search(r"\{.*\}", text, re.DOTALL)` to extract the JSON if the model wrapped it in ```` ```json ````.
  - If `json.loads` fails → return 502 plus `raw` (first 500 chars) for debugging.
  - **Normalize**: if the model forgot `overall` → compute the mean of the 4 criteria; if it forgot `winner` → pick the highest overall.
- ➡️ A great example of **"never trust the LLM to return correct format"** — always have a defensive layer.

### `/api/distill_memory` — Long-term memory
- After each session, for each agent: send personality + old memory + conversation, ask it to rewrite the memory (up to 5 bullet points), merging old + new, dropping trivia.
- If the conversation is empty → return the old memory (no pointless LLM call).

### `/api/session/save` and `/api/session/{sid}` — Session sharing
- **save**: check size (`MAX_SESSION_BYTES = 1MB`, anti-abuse) → make a random code `uuid4().hex[:10]` → write a JSON file.
- **get**: **prevent path traversal** with `re.fullmatch(r"[A-Za-z0-9]{1,32}", sid)` — only accept alphanumerics. Not found → 404; read error → 500.

### Serving the frontend
- `@app.get("/")` returns `index.html`.
- `app.mount("/", StaticFiles(...))` serves all static files.

> 💡 **A common trap question:** "Is the session-sharing endpoint secure?" → Answer honestly: the README clearly states there's **no authentication**, anyone with the code can read the session, suitable for local use. To go public it needs auth + rate limiting + TTL (session expiry). **Knowing and naming the limitation** scores more than hiding it.

---

## 4.5. Summary of backend technical decisions

| Decision | Reason |
|----------|--------|
| One shared `llm.py` for all providers | Thanks to OpenAI-compatibility → swapping a provider is just a config change |
| Wrap all LLM errors as `LLMError` | The upper layer handles errors consistently, simply |
| `search.py` never raises | An auxiliary feature must not crash the main flow (graceful degradation) |
| Let the LLM decide whether to search | Save network calls, fetch facts only when truly needed |
| Defensive layer when parsing scoring JSON | The LLM doesn't guarantee a correct format |
| Stateless backend (except shared sessions) | Simple, easy to scale; real state in the frontend |
| Block path traversal in session codes | Basic security for the file-reading operation |
