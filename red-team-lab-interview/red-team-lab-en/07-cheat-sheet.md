# 07 — Cheat Sheet

- **One line:** educational prompt-injection game, seven direct + five RAG levels, deterministic simulator.
- **No default backend:** browser ES modules; serve over local HTTP.
- **Engines:** `runAttack` direct; `runIndirect` poisoned documents; modeled patterns + thresholds + defenses.
- **State:** completion/best attempts in `localStorage`; `primed` across direct chat turns.
- **Real-model sandbox:** optional `llm.js` calls Gemini/Groq/OpenAI/Ollama; no app-server key storage.
- **Run:** `npm run serve` → port 8088; `npm test` → three Node suites.
- **Limit:** simulator success does not prove a prompt works on a real LLM. Never use real secrets.
