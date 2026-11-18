# 04 — Code Deep Dive

| File | Responsibility |
|---|---|
| `levels.js` | `LEVELS`: seven direct levels, dummy secrets, defenses, hints and lessons. |
| `engine.js` | `TECHNIQUES` patterns; `runAttack(level, userText, state)` applies filters, judge, scores, transformations and returns result/state; `checkGuess`. |
| `indirect.js` | `TECHS`, `RAG_LEVELS`: five levels; `runIndirect(level, poison)` models documents, provenance and output scanning. |
| `app.js` | `route()`, `renderLevel`, `renderRag`, `renderSandbox`, progress and scores UI. |
| `store.js` | `store.solved`, `markSolved`, `recordScore`, `reset` on `localStorage`. |
| `llm.js` | `providerList`, `buildSystemPrompt`, `callLLM`, `detectLeak`; only for optional real-model sandbox. |
| `tests/*.mjs` | Node tests for engine, indirect and store; `npm test` invokes all three. |

Trace `runIndirect` in order: instruction filter → detect techniques → provenance → threshold → output scan. Changing order can change a lesson's result; tests should cover both blocked and leaked paths.
