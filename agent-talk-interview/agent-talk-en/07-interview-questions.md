# 7. Interview Question Bank & Sample Answers

60+ questions grouped by topic, easy to hard. Each has a **suggested answer**. Read them, then rephrase in your own words.

> ⭐ = very commonly asked. Prioritize these.

---

## A. Overview & motivation (warm-up)

**A1. ⭐ Briefly introduce your project.**
> Agent Talk is a web app where multiple AI characters with distinct personalities talk and debate in real time, while the user watches and steers. The FastAPI backend orchestrates LLM calls (via an OpenAI-compatible API), supports token-by-token streaming, summarizes when long, does web lookups, scoring, and long-term memory. The frontend is plain JS that handles the turn loop, session saving, and file export.

**A2. Why did you build this?**
> To practice the core problems of building real AI products: orchestrating multiple LLM calls, handling streaming, managing long context, and integrating external services — rather than just calling an API once.

**A3. Who are the users?**
> People curious about AI, people who want to brainstorm/debate from multiple angles, and myself to learn. It leans toward an exploration & technical-showcase tool more than a commercial product.

**A4. How long did it take? Solo or team?**
> (Answer per your reality — clearly state your role and what you built yourself.)

---

## B. Architecture & design

**B1. ⭐ Describe the overall architecture.**
> Two parts: a static frontend (HTML/CSS/JS) running in the browser, and a FastAPI backend. The backend has 4 layers: `config` (config), `llm` (LLM calls), `search` (web lookup), `main` (orchestration + endpoints). The frontend holds the conversation state and calls the REST endpoints; the speaking part uses HTTP streaming.

**B2. ⭐ Where does conversation state live? Is the backend stateful?**
> The backend is almost **stateless** — each `turn_stream` request carries enough context, sent by the frontend. The real state lives in the frontend (`history`, `summary`...) and is saved to localStorage. The only exception is **shared** sessions written as files in `storage/sessions`.

**B3. Pros/cons of this stateless design?**
> Pros: simple server, easy horizontal scaling, no session store. Cons: the payload grows with the conversation (mitigated by summarization), and a session is tied to one browser (partly solved by share links).

**B4. Why split into many small endpoints instead of one big one?**
> Separation of concerns, easier to test/debug; the frontend drives the loop so it can stop/step in at any time; auxiliary tasks (summarize, score) are called only when needed.

**B5. Why does the frontend hold the orchestration loop, not the backend?**
> So the user has direct control: stop, step in, redirect instantly. The backend just serves each "piece". If the backend ran the whole loop, mid-loop interaction would be hard.

**B6. What if you had to move the loop to the backend?**
> You'd need a two-way channel (WebSocket) for the client to control mid-loop, and a server-side session store. More complex but it would allow many people to watch one session in realtime.

---

## C. LLM & Prompt Engineering

**C1. ⭐ What does "OpenAI-compatible" mean and why does it matter here?**
> It's OpenAI's standard API format (`/chat/completions`, body with model/messages/temperature...) that Groq, Gemini, OpenRouter all mimic. So you only write one LLM-calling function and swap base_url/key/model per provider.

**C2. ⭐ What roles does the `messages` array have? How does the project map them?**
> system (rules/context), user (others' words), assistant (the model's words). The project: the speaking agent's words → assistant; others' words → user as "Name: content". So the LLM stays in character.

**C3. What is temperature? Which values does the project use and why?**
> Randomness/creativity. Dialogue uses ~0.9 (needs variety); summary/scoring/speaker-pick/query-thinking use 0.2-0.3 (needs accuracy, consistency).

**C4. How does an agent "know" who it is and what role to play?**
> Via the system prompt: "You are {name}. {personality}..." plus rules (concise 2-4 sentences, full Vietnamese diacritics, no action descriptions, don't play others), plus the role mapping in the history.

**C5. How does the "hidden goal" work?**
> Adds a private agenda to the system prompt with the instruction "don't reveal it, subtly lead". The LLM tries to pursue the goal discreetly.

**C6. How do you force the LLM to return JSON when scoring? What if it returns malformed output?**
> The prompt demands "return only JSON in the form {...}, nothing else". Defenses: regex-extract `{...}`, return 502 if parsing fails, auto-compute overall/winner if missing. Principle: never trust the LLM to return the right format.

**C7. What is hallucination? How does the project reduce it?**
> When the LLM "makes things up". The project reduces it via web lookup (simple RAG) injecting real evidence into context, plus a "don't make things up" instruction and source footnotes for verification.

---

## D. Streaming & Async

**D1. ⭐ Explain the end-to-end streaming flow.**
> The LLM sends chunks (`data: {...delta...}` lines) → `chat_stream` (async generator) yields the delta → `StreamingResponse` pushes it back → the frontend uses `response.body.getReader()` to read and append progressively to the chat bubble.

**D2. What is async/await? Why is it needed here?**
> It lets the program yield control while waiting on I/O (a network call), so the server serves many requests at once without blocking. Important because calling the LLM takes several seconds.

**D3. What is an async generator?**
> An `async def` function with `yield`, which produces data progressively and can `await` inside. Used to stream text chunks from the LLM.

**D4. Why have both `chat` and `chat_stream`?**
> `chat_stream` for dialogue (typing UX). `chat` (non-stream) for internal tasks that need the whole result at once (summarize, score, pick speaker).

**D5. How is streaming different from WebSocket? Why not use WebSocket?**
> HTTP streaming is one-way server→client, simple. WebSocket is two-way, fits continuous realtime. Here each turn is a separate request, so HTTP streaming is enough and leaner.

**D6. What happens if the user hits Stop mid-stream?**
> The frontend sets `running=false`, the read loop detects it and calls `reader.cancel()` to abort, and the empty bubble is removed.

---

## E. Context management

**E1. ⭐ What is the context window? What problems do long conversations cause?**
> It's the token limit the LLM sees per call. Long conversations may exceed the limit, get slow, and cost more.

**E2. ⭐ How does the project handle long conversations?**
> Rolling summary: keep the last 8 turns verbatim; when exceeding 16 turns, compress the older ones into a summary (merged cumulatively), then only send summary + last 8 turns.

**E3. Why summarize on the frontend instead of automatically on the backend each turn?**
> Because the frontend holds the state and decides when it's needed. The backend offers `/api/summarize` as a pure service, called when the frontend sees it's long enough.

---

## F. Web lookup

**F1. ⭐ How does the lookup feature work?**
> Before speaking, ask the LLM whether it needs real facts and to give one query; if so, search (Tavily if a key, else DuckDuckGo), inject the results into context, and the agent speaks with source footnotes.

**F2. Why let the LLM decide whether to search?**
> Saves network calls and time; fetches facts only when truly needed for a strong argument.

**F3. What if search errors out/times out?**
> `search.py` swallows all errors and returns an empty list → the agent speaks normally (graceful degradation). An auxiliary feature must not crash the main flow.

**F4. What's the weakness of scraping DuckDuckGo with regex?**
> It's brittle — if they change the HTML, the regex breaks. It's a trade-off for free, no-key search. A more robust option is Tavily or an official search API.

**F5. Is this RAG?**
> Yes, a simple form of RAG: pulling external data into context to answer more accurately.

---

## G. Security

**G1. ⭐ How are API keys protected?**
> Stored in `.env` (gitignored), read only on the backend. The `/api/config` endpoint only returns `configured: true/false`, never the real key to the frontend.

**G2. ⭐ What's the risk with the session-sharing endpoint?**
> No authentication — anyone with the code can read the session. Suitable for local use. Public deployment needs auth, rate limiting, and a TTL expiry. (Naming this limitation is a plus.)

**G3. Where is path traversal prevented?**
> In `/api/session/{sid}`: only accept codes matching `[A-Za-z0-9]{1,32}`, blocking paths like `../`.

**G4. How is XSS prevented?**
> The frontend `escapeHtml()`s all dynamic content (names, dialogue, source titles) before putting it into innerHTML; source links must match `^https?://` to be clickable.

**G5. Is there a session size limit?**
> Yes, `MAX_SESSION_BYTES = 1MB`, exceeding it returns 413 — anti-abuse for storage.

---

## H. Testing

**H1. ⭐ How do you test without spending API keys?**
> With pytest and **mocking** the LLM-calling function (`monkeypatch` replaces `llm.chat`/`chat_stream` with a fake). So tests run offline, fast, free, and stable.

**H2. What do you test?**
> The endpoints (`/api/config`, summarize, next_speaker, score, distill_memory, turn_stream, session) and the pure functions in `search` (format_results, _clean). Including error branches: empty history, malformed JSON, invalid session code.

**H3. Why mock the LLM instead of calling it for real?**
> The LLM returns random output (hard to assert), costs money, needs the network, may rate-limit. Mocking gives deterministic tests and isolates the orchestration logic from external services.

**H4. An interesting example test?**
> `test_score_normalizes_overall_and_winner`: mock the model to return JSON missing `overall` and `winner`, then verify the backend auto-computes the mean (e.g. (8+6+7+9)/4 = 7.5) and picks the right winner.

---

## I. Deployment & operations

**I1. How do you run it locally?**
> `pip install -r requirements.txt`, create `.env` with the API key, then `uvicorn backend.main:app --reload`, open `127.0.0.1:8000`.

**I2. How does Docker package it?**
> `python:3.13-slim` image, install requirements first (to leverage layer caching), copy backend+frontend, run uvicorn on `0.0.0.0:8000`, pass the API key via an environment variable at `docker run`.

**I3. Why install dependencies before copying code in the Dockerfile?**
> To leverage Docker layer caching: code changes often but dependencies rarely, so putting the install step first makes rebuilds faster.

---

## J. Open-ended / thinking

**J1. ⭐ The hardest challenge in the project?**
> (Hint) Making end-to-end streaming smooth and stoppable mid-stream; or handling the LLM's non-conformant JSON when scoring. Tell the problem → how you reasoned → the result.

**J2. ⭐ What would you improve with more time?**
> Add auth + TTL for shared sessions; replace DuckDuckGo scraping with a stable search API; sync sessions/memory across devices; let the audience vote alongside the AI judge; add rate limiting.

**J3. Where would the project "break" under many concurrent users?**
> Each LLM call is slow and may hit provider rate limits; you'd need a queue/concurrency limit and rate limiting. Storing sessions as files isn't ideal at scale either — move to a DB/object storage.

**J4. What if the LLM provider changes its API format?**
> Since it's abstracted in `llm.py`, you only fix one place. If the format differs a lot from the OpenAI standard, write a dedicated adapter for that provider.

**J5. How do you keep agents from repeating each other or going off-topic?**
> Prompt rules (concise, in-character), the speaker-selection that follows the flow, the director stepping in, and summaries keeping the main thread. Could improve further by periodically re-stating the topic.

**J6. What's the project's biggest trade-off?**
> Pushing orchestration to the frontend (simpler server, good interaction) at the cost of large payloads and difficulty with many-viewers-one-session realtime. And choosing vanilla JS (lightweight, no build) at the cost of harder-to-scale complex UI.

---

## K. "Specific code" questions (you may be quizzed)

**K1. How does `build_messages` handle the no-history case?**
> It adds a primer user message "Start the conversation [about topic...]" so the LLM has something to respond to.

**K2. Why does it sometimes add "(continue the conversation)"?**
> If the last message is the assistant role (the agent's own words), a user message is needed for the LLM to continue — so it inserts this.

**K3. What does `get_provider` return if the id doesn't exist?**
> Falls back to the `default` provider (never None) → safe.

**K4. What is the `X-Sources` header for and how is it encoded?**
> It sends the list of web sources (title+url) to the frontend with the stream response; it's `quote`d (URL-encoded) for character safety, and the frontend `decodeURIComponent`s then JSON-parses it.

**K5. When sharing a link, why is `memory` stripped?**
> Memory is an agent's private data across sessions; it shouldn't be sent to the link's recipient.

---

## 🎤 The 5 most common questions — rehearse these first
1. A1 — Introduce the project.
2. D1 — The end-to-end streaming flow.
3. C2 — The messages role mapping.
4. E2 — Handling long conversations (rolling summary).
5. C6 — Forcing & defending JSON when scoring.

## ❓ Questions YOU should ask the interviewer (shows initiative)
- Which LLM provider does the team use, any cost/latency concerns?
- How does the team evaluate LLM output quality (eval)?
- Does the product use streaming/RAG, and what's the current architecture?
