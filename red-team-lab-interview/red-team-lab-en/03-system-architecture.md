# 03 — System Architecture

```text
index.html → app.js (hash router, levels, UI, progress)
  ├─ levels.js + engine.js: 7 direct levels, runAttack/checkGuess
  ├─ indirect.js: 5 poisoned-document/RAG levels, runIndirect
  ├─ store.js: completion and best scores in localStorage
  ├─ i18n.js / audio.js: VI/EN and sound
  └─ llm.js: optional sandbox → Gemini/Groq/OpenAI or local Ollama
```

`app.js` reads the hash route, chooses a direct/RAG/sandbox/scores screen and invokes the relevant engine. `engine.js` and `indirect.js` run entirely in the browser, with no backend call. `llm.js` calls a provider directly only if the user chooses real-model mode; the key remains in page memory, which is still risky on an untrusted browser/device.

In seven direct levels the attacker writes chat messages; in five RAG levels they write a **document** for simulated retrieval. Both modes teach mechanisms rather than establish actual LLM jailbreak rates.
