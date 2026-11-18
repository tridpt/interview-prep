# 02 — Foundational Concepts

## Prompt injection and trust boundaries

Prompt injection tries to promote lower-priority text (chat input or retrieved content) into higher-priority instructions. In the five RAG levels, the retrieved document is **untrusted data**; it must not change the assistant's policy or gain access to secrets.

## Defenses as modeled in the game

- **Input filter:** keyword blocklist, easily bypassed by different wording.
- **Output filter/scan:** matches the raw secret; encoding/transformation may evade this string-based simulator.
- **System strength/judge:** raise a threshold or block detected patterns; the simulated judge is not an actual LLM.
- **Provenance/trust boundary:** treat instructions inside a document as data. This is still a modeled rule, not a formal security guarantee.

## Deterministic versus real models

`runAttack` and `runIndirect` use regex patterns, scores and thresholds for repeatable lessons. Real models have different policies and variable behavior; the optional `llm.js` sandbox sends requests to a provider, but game scores are not production security measurements.
