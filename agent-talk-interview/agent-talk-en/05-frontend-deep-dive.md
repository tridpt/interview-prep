# 5. Frontend Deep Dive (explaining `app.js`)

The frontend is **vanilla JavaScript** (no framework), almost entirely in `frontend/app.js`. It's the "brain" driving the conversation loop on the client side.

---

## 5.1. Global state variables

```javascript
let providers = [];        // provider list fetched from /api/config
let running = false;       // whether the loop is running
let history = [];          // [{name, text, kind, sources}] — the whole conversation
let lastSpeaker = "";      // the most recent speaker
let summary = "";          // the summary of older turns
let summarizedCount = 0;   // up to which turn has been summarized
```

Important constants:
- `SUMMARY_WINDOW = 16`: beyond this, start summarizing.
- `KEEP_RECENT = 8`: always keep the last 8 turns verbatim.
- `SS_KEY = "agentTalkSession"`: the localStorage key for the session.
- `LB_KEY = "agentTalkLeaderboard"`: the key for the leaderboard.

---

## 5.2. Initialization — `init()`

```javascript
async function init() {
  await loadProviders();              // get providers from the backend
  const fromShare = await loadFromShare();   // prefer opening a share link
  if (!fromShare && !restoreSession())       // else restore the old session
    DEFAULT_AGENTS.forEach(addAgentCard);    // else → 2 default agents
  bindEvents();
  refreshUi();
  renderLeaderboard();
}
```
Data source priority: **share link → localStorage session → defaults**.

---

## 5.3. Agent card management — `addAgentCard()`, `getAgents()`

- `addAgentCard()`: clones an agent's HTML template, fills the provider dropdown, fills the persona-template dropdown, wires events (pick template, remove, save on type). Each agent gets a **color** by index (`COLORS`).
- `getAgents()`: reads back from the DOM into an array of `{name, persona, provider, goal, memory}` to send to the server.
- Constraint: minimum 2 agents (blocks removal if only 2 remain).

---

## 5.4. The main loop — `loop()` (study this carefully)

```javascript
async function loop(firstSpeaker) {
  running = true;
  const maxTurns = parseInt($("maxTurns").value) || 12;
  let speaker = firstSpeaker;
  for (let i = 0; i < maxTurns && running; i++) {
    await maybeSummarize();                       // compress context if long
    const target = newBubble(speaker);            // create an empty bubble
    const { text, sources } = await streamTurn(speaker, target);  // stream text in
    if (!text) { ...; break; }                    // no content → stop
    renderSources(bubbleEl, sources);             // show source footnotes
    history.push({ name: speaker, text, kind: "agent", sources });
    saveSession();
    tts.speak(speaker, text);                     // read aloud if on
    speaker = await pickNext(speaker);            // choose the next speaker
    if (delayMs) await sleep(delayMs);            // pause between turns
  }
  await updateMemories();                         // update memory at session end
  stop();
}
```

**Things to grasp:**
- The loop stops when: `maxTurns` is reached, the user hits Stop (`running=false`), or an agent returns empty.
- `running` is checked continuously to allow a **smooth mid-loop stop** (even mid-stream — see `streamTurn`).

### `streamTurn()` — Reading the stream from the server
```javascript
const resp = await fetch("/api/turn_stream", {... body: JSON.stringify({...})});
const reader = resp.body.getReader();
const dec = new TextDecoder();
let full = "";
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  full += dec.decode(value, { stream: true });   // append chunk
  target.textContent = full;                      // show progressively
  scrollChat();
  if (!running) { reader.cancel(); break; }       // stop mid-stream
}
```
- Reads `X-Sources` from the header for web sources.
- `decodeURIComponent` to decode the header (the server `quote`d it).

### `pickNext()` — Choose the next speaker
- `auto` mode + >2 agents → call `/api/next_speaker` (the LLM decides).
- Otherwise (or on error) → **round-robin**: the next person in the list.

---

## 5.5. Debate mode — `debateLoop()`

Differs from `loop()` in that it runs through **3 fixed phases** for every agent:
```javascript
const DEBATE_PHASES = [
  { key: "mo_man",    label: "Mở màn" },   // Opening
  { key: "phan_bien", label: "Phản biện" },// Rebuttal
  { key: "ket_luan",  label: "Kết luận" }, // Closing
];
```
- Each phase: show a banner → each agent speaks (passing `phase.key` so the backend changes the prompt).
- After the debate → **auto-scoring** (`scoreConversation()`).

---

## 5.6. Client-side summarization — `maybeSummarize()`

```javascript
if (history.length - summarizedCount <= SUMMARY_WINDOW) return;  // not needed yet
const upTo = history.length - KEEP_RECENT;       // how far to summarize (keep last 8)
const chunk = history.slice(summarizedCount, upTo);
// call /api/summarize with prev_summary + chunk
summary = j.summary; summarizedCount = upTo;
```
After summarizing, `streamTurn` only sends `history.slice(summarizedCount)` + `summary` → keeps the payload small.

---

## 5.7. Director — `interject()`

The user types in the director box → pushes a turn `{name: "Đạo diễn", kind: "director"}` into history. That turn appears in the history sent to the server next turn (as `user`), so agents "hear" it and react.

---

## 5.8. Save & restore session (localStorage)

- `sessionData()`: gathers all state (agents, config, history, summary...) into an object.
- `saveSession()`: writes to localStorage (called constantly on any change).
- `restoreSession()`: reads it back on page load.
- `forgetSession()`: clears the saved session.

---

## 5.9. Share link — `shareLink()` & `loadFromShare()`

- `shareLink()`: sends the session to `/api/session/save`, gets a code, builds a URL like `...#s=<code>`, copies to clipboard.
  - **Privacy/security note:** `getAgents().map(({ memory, ...rest }) => rest)` → **strips the `memory` field** before sharing (private memory isn't sent).
- `loadFromShare()`: reads the URL hash.
  - `#s=<code>` → short link, fetched from the server.
  - `#share=<base64>` → old link (embeds the whole session in the URL).
  - After loading → `replaceState` removes the hash so a reload doesn't reload it.

---

## 5.10. Scoring & leaderboard

- `scoreConversation()` → calls `/api/score` → `renderScore()` draws the per-criterion bars + the winner (👑).
- `recordScore()` → accumulates into localStorage (`LB_KEY`): session count, win count, total score.
- `renderLeaderboard()` → a table sorted by wins, then average score. Has medals 🥇🥈🥉.

---

## 5.11. Text-to-Speech — the `tts` object

- Checks `"speechSynthesis" in window` (not every browser supports it).
- `voiceFor(name)`: picks a **stable voice by name** via a simple hash (the same name always gets the same voice), preferring Vietnamese voices.

---

## 5.12. Client-side security: XSS prevention — `escapeHtml()`

```javascript
const escapeHtml = (s) => s.replace(/[&<>"']/g, (c) => ({...}[c]));
```
All user/LLM-generated content (agent names, dialogue, source titles) is **escaped** before being put into `innerHTML` → preventing **XSS** (injecting malicious code via content). Source links are also checked for `^https?://` before being clickable.

> 💡 **An interview point:** being able to say you're aware of XSS and escape dynamic content is a big plus for security thinking.

---

## 5.13. Frontend responsibility summary

| Function group | Job |
|----------------|-----|
| `init`, `loadProviders` | Initialization |
| `addAgentCard`, `getAgents`, `recolor` | Agent management |
| `loop`, `debateLoop`, `streamTurn`, `pickNext`, `cont`, `stop` | Conversation loop |
| `maybeSummarize` | Context compression |
| `interject` | Director stepping in |
| `saveSession`, `restoreSession`, `shareLink`, `loadFromShare` | Save/restore/share |
| `scoreConversation`, `renderScore`, `recordScore`, `renderLeaderboard` | Scoring & ranking |
| `updateMemories` | Long-term memory |
| `save`, `copyConversation`, `buildContent` | Export/copy |
| `tts.*` | Read aloud |
| `escapeHtml`, `renderSources` | Safe display |
