# 06 — Interview Questions

**1. Does this really jailbreak an LLM?** Not in the default game. `engine.js`/`indirect.js` are regex/score/threshold simulators. The optional sandbox can call a chosen real model, but its 12 levels do not measure that model's attack success rate.

**2. Direct versus indirect?** Direct puts attacker text in a chat message; indirect embeds it in retrieved content, crossing the data/instruction trust boundary. They have separate `runAttack` and `runIndirect` engines.

**3. Why is output scanning insufficient?** In this simulator it matches the raw secret; transformed or encoded output can evade string equality. More fundamentally, keep sensitive data out of the reach of untrusted content and tools.

**4. What is provenance?** Tracking source and authority: retrieved documents are data, not instructions. `runIndirect` models blocking some patterns but is not a complete defense proof.

**5. How does multiturn work?** `runAttack` returns `state.primed` and the UI carries it into the next turn. This is not server session memory or an actual model's long-term memory.

**6. Where do provider keys go?** `llm.js` calls providers from the browser in real-model sandbox mode. Keys stay in page memory for that session; use only dummy secrets and understand third-party API risks.

**7. Which tests?** `npm test` runs `engine.test.mjs`, `indirect.test.mjs`, `store.test.mjs` in Node. This is not browser E2E or an audit of real LLM behavior.
