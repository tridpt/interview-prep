# 02 — Foundational Concepts (from zero)

---

## 1. What is a token?

A language model (LLM) doesn't read text letter by letter or word by word, but as **tokens** — "chunks" of text. A token can be a short word, part of a word, a punctuation mark, or whitespace. In English, on average ~4 chars ≈ 1 token; "ChatGPT" might split into "Chat" + "GPT".

**Why it matters?** Providers **bill by tokens** (both input and output). Knowing the count lets you budget and avoid surprises.

## 2. What is a tokenizer?

A **tokenizer** converts text ⇄ a list of tokens (really a list of integers — "token ids"). Each model uses a specific tokenizer; counting with the *wrong* tokenizer gives the *wrong* number.

## 3. BPE (Byte Pair Encoding)

Most modern tokenizers use **BPE**. The idea: start from single characters/bytes, then **merge** the most frequent pairs into larger tokens, repeating until the vocabulary is large enough. Result: common words become one token, rare words split into pieces. BPE balances "compact vocabulary" with "can represent any text".

## 4. tiktoken and "encoding"

**tiktoken** is OpenAI's fast BPE tokenizer library. Each model family maps to an **encoding** (vocabulary + merge rules):

| Encoding | Used for |
|----------|----------|
| `o200k_base` | GPT-4o, GPT-4.1, o3, o4-mini... (newer) |
| `cl100k_base` | GPT-3.5 Turbo, GPT-4 (older) |
| `p50k_base`, `r50k_base` | older GPT-3 models |

In the project, each model preset declares its `encoding`; an unknown model falls back to `o200k_base`.

> 🔑 Note: tiktoken is **OpenAI's** tokenizer. For other vendors' models (Claude, Gemini), tiktoken counts are an **approximation**, not exact. Worth saying plainly in an interview.

## 5. Input / output tokens

- **Input tokens** (prompt): the text you send the model (question, system message, context).
- **Output tokens** (completion): the text the model generates.
- Output is usually **pricier** than input. Total cost ≈ input×price_in + output×price_out.
- Some vendors also separate **cache tokens** (read from cache, cheaper) and **reasoning tokens** (the "thinking" of o-series models). The project aggregates these too.

## 6. Credits vs Tokens

Not every tool exposes tokens. Some (like Kiro on the test machine) only log **credits** (an abstract billing unit) and **context % used**. The app respects this: it reports credits, not fabricated tokens.

## 7. FastAPI — the backend framework

**FastAPI** is a modern Python web framework:
- Declares routes via decorators: `@app.get("/api/health")`, `@app.post("/api/count")`.
- **Auto-validates** request bodies with **Pydantic** (`BaseModel`) → wrong types return a 422 automatically.
- Async support, auto OpenAPI docs.
- Serves static files via `StaticFiles`.

## 8. Pydantic — input validation

```python
class CountRequest(BaseModel):
    model: str = "gpt-4o-mini"
    input_text: str = ""
    output_text: str = ""
```
When a client POSTs `/api/count`, FastAPI parses the JSON into a `CountRequest`, ensuring types and filling defaults. → Safe code, less boilerplate.

## 9. `lru_cache` — memoizing functions

`tiktoken.get_encoding(...)` is a bit costly. The project wraps the encoder lookup with `@lru_cache(maxsize=32)`: the first call for a model computes and **remembers**; later calls for the same model return instantly. → Noticeably faster while typing.

## 10. `dataclass` — lightweight data structures

`usage_scanner.py` uses `@dataclass` to define `UsageRecord` and `SourceScan` — auto `__init__`, easy `asdict()` conversion. A clean Python way to describe a "data record".

## 11. Regex — filtering log lines

Before parsing JSON (expensive), the app uses regex to quickly filter: only lines **containing** keywords like `usage`, `input_tokens`, `meteringEvent`... move on. An important optimization since log files can be huge.

## 12. Recursion — walking the JSON tree

Each AI tool logs with a different nested JSON shape. The app uses a **recursive** `walk()` to descend into every dict/list, finding any object "that looks like usage". Recursion is the natural technique for traversing a tree of unknown depth.

## 13. Debounce — avoid spamming the API

On the frontend, if every keystroke called the API, it would overload. **Debounce**: wait a short time (220ms) after the last keystroke before calling. Implemented with `clearTimeout` + `setTimeout`.

---

## Quick glossary

| Term | One line |
|------|----------|
| Token | A text chunk — the unit a model reads and is billed by |
| Tokenizer | Converts text ⇄ token ids |
| BPE | Merges frequent character pairs into tokens |
| Encoding | A tokenizer vocabulary (o200k_base, cl100k_base) |
| Input/Output token | Prompt sent / text the model generates |
| Credit | Abstract billing unit (not a token) |
| FastAPI | Async Python web framework |
| Pydantic | Validate input via classes |
| lru_cache | Remember function results to skip recomputation |
| dataclass | Concise way to define a data record |
| Debounce | Defer a call until typing stops |
