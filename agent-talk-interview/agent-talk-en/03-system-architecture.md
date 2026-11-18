# 3. System Architecture

## 3.1. High-level diagram

```
┌─────────────────────────────────────────────────────────────┐
│                          BROWSER                              │
│  ┌─────────────┐  ┌──────────┐  ┌──────────────────────────┐ │
│  │ index.html  │  │ style.css│  │ app.js (frontend brain)  │ │
│  │ (UI)        │  │ (styling)│  │ - turn-taking loop        │ │
│  │             │  │          │  │ - session save (localStg) │ │
│  │             │  │          │  │ - read streaming, TTS     │ │
│  └─────────────┘  └──────────┘  └──────────────────────────┘ │
│         personas.js (10 persona templates)                    │
└───────────────────────────┬─────────────────────────────────┘
                            │  HTTP (fetch) — JSON & text stream
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                BACKEND — FastAPI (main.py)                    │
│   ORCHESTRATION layer: the /api/* endpoints                   │
│   turn_stream · next_speaker · summarize · score ·           │
│   distill_memory · session/save · session/{id} · config      │
│        │                          │                           │
│        ▼                          ▼                           │
│   ┌─────────┐               ┌───────────┐                     │
│   │ llm.py  │               │ search.py │                     │
│   │ call LLM│               │ web search│                     │
│   └────┬────┘               └─────┬─────┘                     │
│        │   reads config from config.py (.env)                 │
└────────┼─────────────────────────┼───────────────────────────┘
         │                         │
         ▼                         ▼
  ┌──────────────┐         ┌─────────────────────┐
  │ LLM Provider │         │ DuckDuckGo / Tavily  │
  │ Groq/Gemini  │         │ (web search)         │
  └──────────────┘         └─────────────────────┘
```

## 3.2. Layers & responsibilities (separation of concerns)

| Layer | File | Responsibility | Does NOT do |
|-------|------|----------------|-------------|
| **Config** | `config.py` | Read `.env`, define providers | No network calls |
| **LLM comms** | `llm.py` | Call the LLM API (stream & non-stream), handle errors | Doesn't know conversation logic |
| **Web comms** | `search.py` | Web lookup, clean HTML | Never raises to the outside |
| **Orchestration** | `main.py` | Build prompts, sequence turns, summarize, score, save sessions | No direct network calls (via llm/search) |
| **UI** | `frontend/*` | UI loop, display, client-side session save | Holds no API key |

**Key point to state in the interview:** the backend is a **"thin but smart" orchestration layer**. It contains no AI — it arranges when to call the AI, with what context, and how to handle the result. The intelligence lives in the LLM; **the project's technical value is in how it orchestrates**.

## 3.3. Data flow of a single turn (the most important — learn this well)

This is the flow you should be able to recite. When an agent speaks one turn:

```
1. [Frontend] app.js gathers data: agent list, speaker, topic,
   history (from the summarized point onward), summary, temperature, allow_search.
        │ POST /api/turn_stream
        ▼
2. [Backend main.py] build_messages(): builds the `messages` array:
   - system prompt: "You are {name}. {personality}..." + topic + hidden
     goal + memory + debate phase + reply rules.
   - (if any) system: "Summary of earlier events: ..."
   - history: own words → role assistant; others' words → role user "Name: ..."
        │
        ▼
3. [Backend] maybe_search_context(): IF allow_search:
   - ask the LLM "do you need a lookup? if so, give one short query".
   - if yes → search.web_search() → inject results into messages (position 1).
        │
        ▼
4. [Backend] check the provider has an API key (if not → return 400).
        │
        ▼
5. [Backend] llm.chat_stream(): call the LLM API with stream=true,
   read each "data:" line, extract delta.content, yield it.
   FastAPI wraps it in a StreamingResponse, with an X-Sources header (if sources exist).
        │ text stream flows back
        ▼
6. [Frontend] reads the stream with reader.read(), appends progressively to the
   chat bubble (you see text running out). Reads the X-Sources header for footnotes.
        │
        ▼
7. [Frontend] saves the turn into `history`, saves to localStorage, reads aloud
   (TTS if on), then calls pickNext() to choose the next speaker → repeat.
```

## 3.4. API endpoints (quick reference)

| Endpoint | Method | Function |
|----------|--------|----------|
| `/api/turn_stream` | POST | Generate one turn (streaming) — the central endpoint |
| `/api/next_speaker` | POST | Ask the LLM who should speak next ("auto" mode) |
| `/api/summarize` | POST | Summarize the older conversation to compress context |
| `/api/score` | POST | AI judge scores 4 criteria, picks a winner |
| `/api/distill_memory` | POST | Distill the "long-term memory" for an agent after a session |
| `/api/session/save` | POST | Save a shared session, return a short code |
| `/api/session/{sid}` | GET | Read back a shared session by code |
| `/api/config` | GET | Return provider info (safe, no key leaked) |
| `/` and `/...` | GET | Serve the frontend (index.html + static files) |

## 3.5. Why split into many small endpoints instead of one big one?

A common interview question. Answer:
- **Separation of concerns**: one endpoint per job → easier to test and debug.
- **Frontend drives orchestration**: the conversation loop lives on the client, the server just "serves each piece". This lets the user stop, step in, or redirect at any time.
- **Flexibility**: summarize/score/memory are independent tasks, called when needed, not mandatory.

## 3.6. Where does the state live?

Important: **the backend is almost stateless**, except for the shared-session files in `storage/sessions`.
- Each `turn_stream` request **carries enough context** (agents, history, summary) sent up by the frontend.
- **The real state lives in the frontend**: variables `history`, `summary`, `lastSpeaker`... saved to `localStorage`.

➡️ **Upside**: the server is simple, easy to scale horizontally, no complex session store needed.
➡️ **Downside**: the payload grows with the conversation (mitigated by summarization); a session is tied to one browser (partly solved by the share-link feature).
