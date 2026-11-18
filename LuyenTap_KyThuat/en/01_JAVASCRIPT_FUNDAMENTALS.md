# 1. JavaScript Fundamentals

Review the commonly-asked points, tied to real usage in the vanilla-JS projects (cap-cuu-101, quan-quen-map, agent-talk frontend, word-checker web).

## 1. var / let / const & scope
- `var`: function scope, hoisted. **Avoid.**
- `let`/`const`: block scope. `const` can't be reassigned (but object contents can change).
- **Closure**: a function "remembers" variables from its outer scope. (E.g. event handlers in app.js access state via closures.)

## 2. == vs ===
- `==` compares with type coercion (`0 == "0"` → true) → bug-prone.
- `===` strict (type + value). **Always prefer `===`.**

## 3. Async: callback → Promise → async/await
- **Event loop**: JS is single-threaded; I/O runs in the background, results queue up, processed when the stack is empty.
- **Promise**: a future value (`pending → fulfilled/rejected`).
- **async/await**: cleaner syntax for Promises. (E.g. `const res = await fetch(...)` in every frontend.)
- **Reading a stream**: `response.body.getReader()` + a `while` loop reading chunks (agent-talk shows the LLM typing token by token).

## 4. DOM & events
- `document.getElementById/querySelector`, creating elements, `addEventListener`.
- **Event delegation**: attach one listener on a container, use `event.target.closest("[data-...]")` instead of per-element listeners (quan-quen-map uses this a lot).
- **`<template>`**: clone repeated elements with `cloneNode(true)` (agent-talk builds agent cards).

## 5. this, arrow functions
- Arrow functions have **no own `this`** — they take it from the enclosing scope. Useful in callbacks.
- Regular functions: `this` depends on how they're called.

## 6. Common array & object ops
- `map`/`filter`/`reduce`/`find`/`sort`/`some`/`every`.
- Spread `...`, destructuring `{a, ...rest}` (e.g. dropping `memory` when sharing: `({memory, ...rest}) => rest`).
- `JSON.parse`/`JSON.stringify`.

## 7. Browser Web APIs (often asked since the projects use many)
- **fetch + Streams**, **localStorage** (text, ~5MB), **IndexedDB** (large, async — quan-quen-map stores images).
- **Service Worker** (PWA, offline cache), **Canvas**, **Web Audio** (cap-cuu-101 synthesizes sound), **Web Speech** (TTS), **getUserMedia** (camera QR scanning — totp-manager).

## Sample questions
- "Explain the event loop?" → single thread + callback/microtask queues.
- "What's a closure, when to use it?" → a function remembering outer variables; e.g. counters, hiding data.
- "localStorage vs IndexedDB?" → text/sync/5MB vs object/async/large.
- "Why use `===`?" → avoid implicit coercion bugs.
- "How do you read streaming data from a server?" → getReader() + decode each chunk.

> 💡 Always **connect to a project**: "I used IndexedDB in quan-quen-map because base64 images fill the 5MB localStorage."
