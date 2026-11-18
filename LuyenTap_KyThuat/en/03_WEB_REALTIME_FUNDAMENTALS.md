# 3. Web & Real-time Fundamentals

Everyone should review this — HTTP, REST, streaming, PWA, web security. Tied to the projects.

## 1. HTTP basics
- **Request/Response**, methods: GET (read), POST (create/send), PATCH (partial update), PUT (replace), DELETE.
- **Status codes**: 200 OK · 201 Created · 301/302 redirect · 400 bad request · 401 unauthenticated · 403 forbidden · 404 not found · 413 payload too large · 429 too many requests · 500 server error · 502/503 upstream error.
- **Headers** (Content-Type, Authorization: Bearer ...), **body** (usually JSON).
- **Idempotent**: GET/PUT/DELETE repeat to the same result; POST doesn't.

## 2. REST API
- Resources via URLs (`/api/accounts`, `/api/accounts/{id}`), actions via methods.
- Stateless: each request carries enough info (e.g. agent-talk sends the whole history each turn).
- All projects in the pack follow REST: totp-manager, outlook, word-checker (web).

## 3. Streaming & real-time
- **HTTP streaming / SSE**: the server pushes data one-way progressively (agent-talk streams LLM tokens: `data: {...}` lines).
- **WebSocket**: two-way, continuous real-time (when needed? real-time chat, games). One-way streaming → SSE/HTTP is leaner.
- **Polling**: the client re-asks periodically (outlook polls import-job status every 1.5s; totp-manager loads codes every 1s).

## 4. CORS
- A browser mechanism that blocks cross-origin API calls unless the server allows them (`Access-Control-Allow-Origin`).
- Local projects enable `allow_origins=["*"]` (safe because localhost).

## 5. PWA (Progressive Web App)
- **Manifest** + **Service Worker** + HTTPS → installable, works offline.
- **SW lifecycle**: install (precache) → activate (clean old caches) → fetch (intercept requests).
- **Cache strategies**: cache-first (map tiles), network-first (pages), stale-while-revalidate (assets). quan-quen-map uses all 4 by origin; cap-cuu-101 precaches the app shell.
- **Cache versioning**: bump `CACHE_VERSION` so clients get the new build.
- Security note: totp-manager **never caches `/api`** (sensitive data + codes change every second).

## 6. Client-side storage
- **localStorage** (text, sync, ~5MB) vs **IndexedDB** (object/blob, async, large) vs **Cache API** (request/response, SW-managed).

## 7. Web security (often asked)
- **XSS**: injecting code via user content → prevent by escaping before innerHTML (every project has `escapeHtml`).
- **CSRF**, **path traversal** (agent-talk restricts session codes via a regex `[A-Za-z0-9]`), **upload size limits** (word-checker 25MB).
- **HTTPS/TLS**, **encryption-at-rest** (totp-manager), **never leak secrets to the client** (only return `configured: true/false`).

## 8. OAuth 2.0 / authentication (outlook-mail-reader)
- **Access token** (short-lived) + **refresh token** (to get a new access token); **device-code flow** (enter a code in a browser); **scope** (permissions). Tokens rotate → you must store the new one.

## Sample questions
- "SSE vs WebSocket?" → one-way vs two-way.
- "How to cache in a service worker without getting stuck on an old build?" → versioning + network-first for pages.
- "What's XSS, how to prevent?" → escape dynamic content.
- "401 vs 403?" → unauthenticated vs authenticated but no permission.
- "Why not cache /api?" → dynamic/sensitive data.

> 💡 Connect to a project: "I used network-first for HTML so users don't get stuck on an old build, but cache-first for map tiles for speed + offline."
