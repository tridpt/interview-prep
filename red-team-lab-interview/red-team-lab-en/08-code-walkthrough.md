# 08 — Code Walkthrough

## Direct chat

1. `app.js` `route()` selects `level/:id`, gets the configuration from `LEVELS` and builds the UI.
2. On input, `runAttack(level, userText, state)` evaluates the input filter, `TECHNIQUES`, power/threshold and judge.
3. If its rules decide to leak, the engine may transform the dummy secret and apply the output filter; it returns `response`, `blockedBy` and `state.primed` for the next turn.
4. The UI records completion and fewest attempts via `store`.

## Indirect document

1. The RAG route loads `RAG_LEVELS`; the attacker writes document content, not a direct chat message to the victim.
2. `runIndirect()` checks a blocklist, patterns, provenance, threshold and output scan; it returns a modeled answer and `achieved` flag.
3. The blue-team lesson explains trust boundaries; this is the simulator's decision, not an actual model response.

## Optional sandbox

`renderSandbox()` reads defense settings and uses the simulator by default. Real-model mode calls `buildSystemPrompt` and `callLLM` in `llm.js`, then `detectLeak` does string-based response detection. It is not a comprehensive security evaluation.
