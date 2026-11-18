# 2. Python Fundamentals

Tied to the projects' backends (agent-talk, totp-manager, word-checker, outlook-mail-reader are all Python).

## 1. Data types & structures
- `list` / `dict` / `set` / `tuple`. `set` for dedup + O(1) membership (e.g. `activeTypes` sets).
- List/dict comprehensions: `[f(x) for x in xs if cond]`.
- `collections.Counter` (frequency counting — word-checker finds the most common font), `defaultdict`.

## 2. Functions & parameters
- Default args, `*args`/`**kwargs`, **keyword-only** (`def f(*, a)`).
- **Lazy import** (import inside a function) — agent-talk/word-checker import the LLM lazily to avoid requiring the dependency when unused.

## 3. Dataclasses & type hints
- `@dataclass` for compact data structures (e.g. `Issue` in word-checker, `ParagraphInfo`, `Session` in totp-manager).
- Type hints (`def f(x: int) -> str`) aid readability + IDE hints (not enforced at runtime).

## 4. Async (async/await)
- `async def` + `await`; runs on an event loop (agent-talk uses async FastAPI + httpx.AsyncClient).
- **Async generator**: `async def` with `yield` → stream chunks (agent-talk's `chat_stream` yields LLM deltas).
- When to async: **I/O-bound** tasks (network/file). For CPU-bound, async doesn't help.

## 5. Error handling & context managers
- `try/except/finally`; custom exceptions (e.g. `TOTPError`, `VaultLocked`, `WrongPassword`).
- `with open(...) as f:` (auto-close); `with httpx.AsyncClient() as c:`.

## 6. Standard library used in the pack
- `hmac`, `hashlib`, `base64`, `struct` — totp-manager self-implements TOTP.
- `re` (regex) — word-checker textcheck/spelling.
- `json`, `csv`, `pathlib.Path`, `uuid`, `os`, `threading` (outlook's multi-threaded import job).

## 7. Web frameworks
- **FastAPI** (agent-talk, totp-manager, outlook): async, Pydantic request validation, `@app.post(...)`, `StreamingResponse`, `StaticFiles`.
- **Flask** (word-checker): sync routes, `send_file`, `render_template`.
- **Pydantic**: `class XxxRequest(BaseModel)` → auto-validates JSON, returns 422 on type errors.

## 8. Virtual environments & dependencies
- `python -m venv .venv`, `pip install -r requirements.txt`.
- Split `requirements.txt` (runtime) vs `requirements-dev.txt` (test/dev).

## Sample questions
- "list vs tuple vs set?" → mutable/ordered, immutable, dedup + fast lookup.
- "When to use async/await?" → I/O-bound; e.g. calling the LLM takes seconds.
- "What's a decorator?" → a function wrapping another (e.g. `@app.post`, `@dataclass`).
- "What's the GIL?" → Global Interpreter Lock: only one thread runs bytecode at a time → threading suits I/O-bound, CPU-bound should use multiprocessing.
- "Why self-write TOTP with hmac/hashlib?" → fewer dependencies, understand the standard, verify with test vectors.

> 💡 Connect fundamentals to a project: "I used an async generator in agent-talk to stream tokens from the LLM to the browser."
