# 01 — Project Overview

**30-second pitch:** Red-Team Lab is a browser-based security education game for prompt injection and defenses. Its main engine is a **deterministic rule simulator**: seven direct-chat levels and five poisoned-document/RAG levels. A configurable sandbox can optionally call a real LLM with the user's key.

## User experience

- Try inputs, inspect detected techniques and why a defense blocked or leaked a dummy secret, then read a blue-team lesson.
- In indirect levels, write a document a simulated assistant retrieves; the document is lower-trust data, not an actual system prompt.
- Track fewest attempts in `localStorage`, switch VI/EN and toggle sound.

## Run and verify

`npm run serve` (or `python -m http.server 8088`), then open `http://localhost:8088`. ES modules need HTTP. `npm test` runs Node tests for engine, indirect and store. The default game needs no backend, installed dependencies or API key.

**Limitation:** simulator outcomes only reflect its own regex/weights/thresholds, not success rates against actual models. Real LLM mode is optional in the sandbox and needs a provider key/network (or local Ollama). Never use real secrets.
