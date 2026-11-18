# 05 — Frontend Deep Dive (part by part)

The frontend is vanilla HTML/CSS/JS, no framework. Files: `index.html`, `app.js`, `style.css`.

---

## A. `index.html` — layout

The UI is split into blocks:
1. **Topbar**: title + "Scan Local Logs" button + status box (`#status`).
2. **Metric grid** (4 big cards): Total tokens, Input tokens, Output tokens, Kiro credits.
3. **"Sources" panel**: list of scanned sources (Claude/Kiro/Antigravity) with status badges.
4. **"Recent usage records" panel**: a table (Time, Source, Model, Input, Output, Total, Credits, Context).
5. **"Accuracy" panel**: an honest note about data limits.
6. **"Paste API usage JSON" panel**: a paste box + Parse button + 3 result cells (Input/Output/Total).
7. **"Manual tokenizer estimate" panel**: model select, 2 Input/Output boxes, token numbers + encoding.

> Note: the static copy is written in accent-free Vietnamese (e.g. "Doc usage thuc te...") — a project quirk that doesn't affect logic.

## B. `app.js` — core logic

### DOM refs + utilities
- All `getElementById` grouped at the top.
- `formatNumber`: uses `Intl.NumberFormat("en-US")` for comma-separated numbers.
- `formatTime`: `new Date(value).toLocaleString()`, with a fallback if unparseable.
- `escapeHtml`: **XSS protection** — replaces `& < > " '` with entities before inserting into `innerHTML`. (A notable security point: log data is untrusted.)

### `api(path, options)` — fetch wrapper
```js
const res = await fetch(API + path, { headers: {"Content-Type":"application/json"}, ...options });
const data = await res.json().catch(() => ({}));
if (!res.ok) throw new Error(data.detail || "Request failed");
return data;
```
Centralizes error handling and headers.

### `scanUsage()` + `renderScan(data)`
- `scanUsage`: set status "Scanning", call `/api/usage/scan`, render, reset status; catch errors to show.
- `renderScan`:
  - Update the 4 totals from `data.totals`.
  - Build each **source card** (name, `status` badge, note, paths, file/record counts).
  - Build the **records table** (up to 250 rows to keep the DOM light), each row's `title = record.path` for hover.
  - Toggle the "No usage records" message by record count.

### `loadModels()` + `countTokens()`
- `loadModels`: call `/api/models`, fill the `<select>`, default `gpt-4o-mini`.
- `countTokens`: update char counts immediately, call `/api/count`, show input/output/total tokens + encoding. On error, show the message in the encoding slot.

### `scheduleCount()` — debounce
```js
clearTimeout(debounceTimer);
debounceTimer = setTimeout(countTokens, 220);
```
Each keystroke cancels the old timer and sets a new 220ms one → calls the API only when typing stops. Avoids request spam.

### Parsing usage JSON (runs on the **client**)
A JS mirror of the backend logic to parse quickly in the browser:
- `looksLikeUsage(value)`: does the dict have any usage key.
- `findUsage(value)`: **recursively** find the usage object (prefer `value.usage`, `value.usageMetadata`, then walk children).
- `pickNumber(obj, keys)`: take the first valid number across a key list.
- `parseUsageJson()`: parse text, find usage, extract input/output/total (total = taken directly or input+output), show; on error show `usageError`.

> Nice point: the usage-detection logic is **duplicated** on backend (Python) and frontend (JS) with the same "try many keys + recurse" philosophy. In an interview you can discuss this trade-off (convenient for the client but must keep two places in sync).

### Event binding + bootstrap
```js
loadModels()
  .then(() => Promise.all([countTokens(), scanUsage()]))
  .catch(...);
```
- Scan/Count/Clear/Parse buttons, model change, input/output typing all have listeners.
- The paste-JSON box also debounces 220ms.
- On load: load models → simultaneously count (empty) and scan logs.

## C. `style.css`
- Light theme, CSS variables (`--bg`, `--panel`, `--blue`...), soft shadows.
- 4-column metric grid, horizontally scrollable records table on narrow screens.
- Status badges colored by `ok` (green) / `no_usage` (amber) / `missing` (red).
- Responsive: 2 columns then 1 on small screens.

## D. Frontend talking points
1. **`escapeHtml` for XSS**: log data is untrusted, so escape HTML before rendering — good security thinking.
2. **Debounce**: smooth UX, no backend spam while typing.
3. **250-row render cap**: keeps the DOM light even with thousands of records.
4. **Clear state separation**: the `#status` box toggles `data-kind` (ok/error) for CSS coloring.
5. **No framework**: shows command of plain DOM APIs, fetch, async/await.
