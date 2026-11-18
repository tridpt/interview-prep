# 6. Deep Analysis of the "Point-Scoring" Features

This file helps you **tell an impressive story** about the complex features. Each one comes with: the idea, how it's done, and "why it's hard/cool".

---

## 6.1. Token-by-token streaming ⭐ (the most important feature)

**Idea:** the user sees the agent "typing" like a real person, not a blank screen.

**The technical chain end-to-end:**
```
LLM (stream=true) → llm.chat_stream (async generator, yields delta)
  → StreamingResponse (FastAPI) → HTTP text stream
  → fetch + response.body.getReader() (frontend) → append & display progressively
```

**Why it's cool:** this is a true **end-to-end streaming** flow, touching async generators, ASGI streaming responses, and the browser's Streams API. Explaining this whole chain shows you understand both backend and frontend.

**Possible follow-ups:**
- *"Why not WebSocket?"* → One-directional streaming (server→client) is simple and enough with HTTP streaming/SSE; WebSocket fits continuous two-way realtime. Here each turn is a separate request, so HTTP streaming is leaner.
- *"How do you stop mid-stream?"* → The frontend checks the `running` flag and calls `reader.cancel()` to abort reading the stream.

---

## 6.2. Multi-agent orchestration & next-speaker selection ⭐

**Three speaking-order modes:**
1. **Round-robin**: simple, take the next person in the list.
2. **AI picks** (`/api/next_speaker`): send the flow to the LLM, ask "who should speak next" → creates a natural-feeling conversation. Has a **round-robin fallback** if the LLM returns an unknown name.
3. **Debate**: 3 fixed phases (opening → rebuttal → closing), each changing the prompt.

**Why it's cool:** shows **orchestration** thinking — turning many separate LLM calls into one coherent conversation. The fallback shows defensive thinking (the LLM can return wrong answers).

---

## 6.3. Role mapping so the LLM "stays in character" ⭐

**A subtle problem:** the LLM API has only 3 roles: system/user/assistant. But we have many agents. How do we make agent A "think" it is A?

**The solution:** when building the prompt for agent A:
- A's own lines in the history → assigned the `assistant` role (the LLM understands "these are my words").
- Everyone else's lines → assigned the `user` role, with the name `"Name: content"`.

➡️ The LLM is reasonably "tricked" into believing it is A talking to the others. **This is the most subtle detail of the prompt design.**

---

## 6.4. Managing long context with a rolling summary ⭐

**The problem:** long conversations exceed the context window, get slow, cost more.

**The solution:** keep the last 8 turns verbatim; when exceeding 16 turns → compress the older ones into a summary (merged with the previous summary). Each new turn only sends: `summary + last 8 turns`.

**Why it's cool:** this is a technique genuinely used in commercial LLM products. Knowing the context-window concept + the compression approach is a big plus.

---

## 6.5. Conditional web lookup (the LLM decides) ⭐

**A smart 2-step flow:**
1. Before speaking, ask the LLM: "Do you need real facts? If so, give one short query; if not, reply NO."
2. If yes → search → inject results into context → the agent speaks with evidence + source footnotes.

**Why it's cool:**
- It doesn't always search (saving cost, faster) — **the LLM decides**.
- This is a simple form of **RAG (Retrieval-Augmented Generation)**: pulling external data to support the answer.
- `search.py` **never raises** → an auxiliary feature doesn't crash the main flow (graceful degradation).

**Possible question:** *"What is RAG?"* → A technique where the LLM "looks up" an external data source and feeds it into the context to answer more accurately, reducing "making things up" (hallucination).

---

## 6.6. AI judge scoring + handling LLM's non-conformant JSON ⭐

**Idea:** a separate LLM call acts as a neutral judge, scoring 4 criteria (logic/evidence/creativity/attitude) and picking a winner.

**The notable technical part — defensive parsing:**
- Use regex to extract the `{...}` block in case the model wraps it in ```` ```json ````.
- If parsing fails → return 502 + raw for debugging.
- If the model **forgets** `overall` → compute the mean; forgets `winner` → pick the highest score.

**Why it's cool:** demonstrates the golden rule when working with LLMs: **"never trust the LLM to return the right format"** — always have a normalization/defensive layer. This is very "battle-tested".

---

## 6.7. Long-term memory

**Idea:** after each session, each agent "distills" what it learned into a few bullet points, stores it, and feeds it into the prompt next session → the agent appears to "remember" and "grow".

**How:** `/api/distill_memory` sends personality + old memory + conversation → the LLM rewrites the memory (merge old+new, drop trivia, max 5 lines).

**Privacy:** when sharing a link, private memory is stripped (not sent).

---

## 6.8. Hidden goal

Each agent can be given a private agenda ("steer the conversation toward X"), told "don't reveal it, subtly lead". It adds dramatic depth to the conversation — and is a fun experiment on the LLM's ability to "role-play with a goal".

---

## 6.9. Multi-provider — a different "brain" per agent

Thanks to OpenAI-compatibility, you can configure 2 providers (e.g. Groq + Gemini) and have each agent use a different model → comparing the "personalities" of the models. The backend just swaps `base_url/key/model`.

---

## 6.10. "Feature ↔ technical concept" table (memorize this)

| Feature | Technical concept you demonstrate |
|---------|-----------------------------------|
| Token-by-token streaming | async generator, StreamingResponse, Streams API, SSE |
| Next-speaker selection | orchestration, defensive fallback |
| Role mapping | prompt engineering, the LLM messages model |
| Rolling summary | context-window management, sliding window |
| Conditional web lookup | RAG, graceful degradation, scraping |
| Scoring JSON | handling untrusted LLM output, validation |
| Long-term memory | distillation, data privacy |
| Multi-provider | abstraction via a common standard, separate config |
| Share link | stateless server, path traversal prevention |
| escapeHtml | client-side XSS security |

> 💡 **Storytelling strategy:** if asked "which feature are you most proud of?", pick **end-to-end streaming** or **defensive JSON handling in scoring** — both show technical depth and battle-tested thinking.
