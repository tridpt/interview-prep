# 8. Cheat Sheet — Quick Review Before Go-Time

A one-pager. Skim for 5-10 minutes right before the interview.

---

## 🎯 30-second pitch (memorize)
> Agent Talk: multiple AIs with distinct personalities talk/debate in real time, the user watches and steps in. A **FastAPI** backend orchestrates **LLM** calls (OpenAI-compatible), **streams** token by token, summarizes when long, does web lookup, scoring, long-term memory. A **vanilla JS** frontend handles the turn loop, session save, file export.

## 🧱 Tech stack
- **Backend**: FastAPI, Uvicorn (ASGI), httpx (async), python-dotenv, Pydantic.
- **Frontend**: plain HTML/CSS/JS; Fetch+Streams, localStorage, SpeechSynthesis, Clipboard.
- **LLM**: Groq (default) / Gemini / OpenRouter — OpenAI-compatible.
- **Search**: DuckDuckGo (free) / Tavily (key).
- **Test**: pytest (mocked LLM). **Deploy**: Docker (python:3.13-slim).

## 🗂️ 4 backend layers
`config.py` (.env config) → `llm.py` (LLM calls, stream/non-stream) → `search.py` (web, never raises) → `main.py` (orchestration + endpoints).

## 🔌 Main endpoints
| Endpoint | Job |
|---|---|
| `turn_stream` | generate 1 turn (streaming) ⭐ |
| `next_speaker` | LLM picks the next speaker |
| `summarize` | compress context |
| `score` | score 4 criteria + winner |
| `distill_memory` | long-term memory |
| `session/save` & `session/{id}` | share a session |
| `config` | provider info (no key leaked) |

## 🌊 Streaming flow (recite this)
LLM (`data: delta`) → `chat_stream` (async generator `yield`) → `StreamingResponse` → frontend `body.getReader()` → append & display progressively.

## 🎭 Role mapping (the subtle detail)
The speaking agent's words → `assistant`; others' words → `user` "Name: content"; context/rules → `system`. → the LLM stays in character.

## 🧠 Context management
Keep the last **8 turns** verbatim; beyond **16 turns** → summarize the older ones cumulatively (rolling summary). Send: summary + last 8 turns.

## 🌡️ temperature
~0.9 for dialogue (creative); 0.2-0.3 for summary/scoring/speaker-pick/query (accurate).

## 🔎 Web lookup (simple RAG)
The LLM decides if a search is needed → search → inject context → source footnotes. Search error → return empty, agent still speaks (graceful degradation).

## 🏆 Scoring — defensive JSON
Regex-extract `{...}`; parse fail → 502; missing `overall`/`winner` → auto-compute/auto-pick. **Never trust the LLM to return the right format.**

## 🔐 Security (remember 4 points)
1. API key only on the backend; `/api/config` returns `configured`, not the key.
2. Path traversal prevention: session code `[A-Za-z0-9]{1,32}`.
3. XSS prevention: `escapeHtml` all dynamic content; links must be `https?://`.
4. Session size limit 1MB. **Known limitation:** shared sessions have no auth → local-only.

## 🧪 Testing
pytest + **mocked LLM** (monkeypatch) → offline, free, deterministic. Tests error branches too (empty history, bad JSON, bad code).

## ⚖️ Main trade-offs
- Backend **stateless**, orchestration on the **frontend** → simple server, good interaction; at the cost of growing payloads, hard multi-viewer realtime.
- **Vanilla JS** → lightweight, no build; at the cost of harder-to-scale large UI.

## 🚀 Future improvements (if asked)
Auth + TTL for shared sessions; a stable search API instead of scraping; cross-device sync; rate limiting; an LLM-call queue under load.

## 💬 Key terms
- **LLM / token / context window**
- **OpenAI-compatible** (swap provider = swap config)
- **streaming / SSE / async generator / ASGI**
- **orchestration / prompt engineering**
- **RAG / hallucination / graceful degradation**
- **stateless / path traversal / XSS / mock**

---

### ✅ Pre-interview mental checklist
- [ ] Can pitch the project in 30 seconds.
- [ ] Can describe one turn's data flow.
- [ ] Can explain end-to-end streaming.
- [ ] Can name 2 technical decisions + reasons.
- [ ] Can name 2 limitations + fixes.
- [ ] Prepared 2 questions to ask the interviewer.

**Breathe. You understand this project. Tell your story with confidence. 🍀**
