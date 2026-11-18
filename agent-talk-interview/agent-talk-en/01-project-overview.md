# 1. Project Overview — Agent Talk

## 1.1. What is Agent Talk?

Agent Talk is a **web application** that lets you create multiple "agents" (AI characters), each with its own **personality (persona)**, and then have them **talk to each other** about a topic. The user is the audience: you watch the conversation unfold in real time, and can "step in" as a **director** to redirect the story.

Think of it as an "AI talkshow": you create 2-3 characters (e.g. an optimist, a skeptic, a philosopher), give them a topic ("Will AI replace humans?"), hit a button, and watch them argue.

### Core points to remember
- This is not a typical human-to-machine chatbot. It is **machine-talking-to-machine**, with the human observing and steering.
- **No GPU needed**, no self-hosted model. It uses **free LLM APIs** (Groq, Gemini, OpenRouter).
- All the "intelligence" comes from the external LLM; the project's own code is a smart **orchestration layer** around that LLM.

## 1.2. What value does the project deliver?

This is more of a **creative and technical-showcase** project than a solution to a specific business problem. Its value:

1. **Exploring LLM behavior**: see how different AI personalities react to the same topic — useful for understanding prompt engineering.
2. **A brainstorming / debate tool**: have two agents debate both sides of an issue to surface angles you hadn't considered.
3. **Entertainment & education**: simulated debate with scoring and a leaderboard.
4. **Demonstrating technical ability**: streaming, multi-turn orchestration, context management, web search integration — this is the part that scores points in an interview.

> 💡 **Interview tip**: When asked "what is this project for", don't just say "for fun". Emphasize the **technical value**: "I built it to practice orchestrating multiple LLM calls, handling streaming, and managing long context — the core problems of building real AI products."

## 1.3. Main features

| Feature | Short description |
|---------|-------------------|
| **Multiple agents** | 2+ agents join a group conversation |
| **Streaming** | Watch each character "type" out in real time |
| **Speaking order** | 3 modes: round-robin / AI picks the next speaker / debate mode |
| **Persona templates** | 10 prebuilt personas + a free-text personality field |
| **Hidden goal** | Each agent has a private agenda to subtly steer the conversation |
| **Director (interject)** | The user steps in mid-conversation to redirect |
| **Auto-summarize** | When the conversation grows long, it compresses to avoid losing context |
| **Text-to-Speech (TTS)** | Browser voices read each agent's lines aloud |
| **Long-term memory** | Agents remember what they learned across sessions |
| **Multi-criteria scoring** | An AI judge scores logic/evidence/creativity/attitude and picks a winner |
| **Cumulative leaderboard** | Aggregates wins & average scores across sessions |
| **Web lookup** | Agents fetch real evidence (free DuckDuckGo or Tavily) before speaking |
| **Copy & share link** | Generate a short link to share a whole session (stored on the server) |
| **Source footnotes** | Lines that used web data include a source link |
| **Save & export** | Auto-saves to localStorage, exports `.txt` / `.md` / `.json` |
| **Multiple "brains"** | Configure 2 providers so each agent can use a different LLM |

## 1.4. Tech stack

**Backend (Python):**
- **FastAPI** `0.137.0` — asynchronous web framework for building the REST API.
- **Uvicorn** `0.49.0` — ASGI server that runs FastAPI.
- **httpx** `0.28.1` — asynchronous HTTP client, used to call the LLM API and web search.
- **python-dotenv** `1.2.2` — reads config (API keys) from a `.env` file.
- **Pydantic** — defines & validates request data (ships with FastAPI).

**Frontend:**
- **Plain HTML + CSS + JavaScript (vanilla)** — no React/Vue. Runs directly in the browser.
- Browser Web APIs: **Fetch API + Streams** (reading the stream), **localStorage** (saving sessions), **SpeechSynthesis** (reading aloud), **Clipboard API**.

**LLM & external services:**
- **OpenAI-compatible** APIs: Groq (default), Google Gemini, OpenRouter.
- Web search: **DuckDuckGo HTML** (free, no key) or **Tavily** (free tier, needs a key).

**Testing & deployment:**
- **pytest** `9.1.0` — automated tests; the LLM is **mocked** so they cost no API key and run offline.
- **Docker** — packaged with the `python:3.13-slim` image.

> 💡 **Why vanilla JS and no framework?** A good answer: "The project's complexity lives in backend orchestration and streaming. The frontend is mostly a state loop and DOM updates, so vanilla JS keeps it **lightweight, build-step-free, and easy to deploy** (just serve static files). If the UI grew more complex, I could switch to a framework later."

## 1.5. How to run (demo flow)

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Create .env and paste an API key (e.g. Groq)
copy .env.example .env

# 3. Run the server
uvicorn backend.main:app --reload

# 4. Open the browser: http://127.0.0.1:8000
```

Usage flow:
1. Set a name + personality for at least 2 agents (you can pick a template).
2. (Optional) enter a topic and a hidden goal per agent.
3. Choose the speaking order, number of turns, speed, creativity (temperature).
4. Hit **Start** → watch them talk. Use the **Director** box to step in.
5. Hit **Continue** for more turns, or save/share/score.

## 1.6. Folder structure

```
agent-talk/
├── backend/
│   ├── main.py      # FastAPI: orchestrates turns, summaries, speaker selection, scoring
│   ├── llm.py       # Calls the LLM (OpenAI-compatible, with streaming)
│   ├── search.py    # Web lookup (Tavily if a key exists, else DuckDuckGo)
│   └── config.py    # Reads provider config from .env
├── frontend/
│   ├── index.html   # Config UI + chat panel
│   ├── app.js       # Conversation loop, session saving, file export
│   ├── personas.js  # Persona template library
│   └── style.css
├── tests/           # pytest tests (LLM is mocked)
├── storage/sessions # Where shared sessions are stored (JSON files)
├── Dockerfile
├── .env.example
├── requirements.txt
└── requirements-dev.txt
```

**Design philosophy:** clear separation — `config` (config) ← `llm`/`search` (external service comms) ← `main` (orchestration logic) ← `frontend` (UI). One responsibility per file (separation of concerns).
