# 4. System Design

For freshers/juniors, design questions are usually light ("design a URL shortener", "design the backend for app X"). What matters is a **clear thinking process**, not memorizing huge architectures.

## A 6-step answer framework
1. **Clarify requirements**: functional (what) + non-functional (how many users, read- or write-heavy, real-time/offline?). **Always ask before drawing.**
2. **Estimate scale**: users, QPS, storage (rough is fine).
3. **API**: list the main endpoints (method + path + in/out).
4. **Data model**: tables/collections, relations, indexes.
5. **Architecture**: client → API → service → DB/cache; where you need a queue, cache, CDN.
6. **Trade-offs & bottlenecks**: consistency vs availability, vertical/horizontal scaling, single points of failure.

## Concepts to know
- **Vertical scaling** = a bigger machine; **horizontal** = more machines + a **load balancer**.
- **Stateless** makes horizontal scaling easy (each request self-contained — like agent-talk).
- **Cache** (Redis/CDN/browser cache) reduces load; **trade-off**: data can be stale.
- **DB**: SQL (relational, ACID) vs NoSQL (flexible, scalable). **Indexes** speed reads.
- **Queue** decouples heavy/slow work to background processing (e.g. LLM calls / bulk email).
- **CAP**: a distributed system can't have consistency + availability + partition tolerance all at once — you choose.

## Tie-ins with the pack's projects (great to mention)
- **agent-talk**: **stateless** backend (state sent by the client each turn) → easy to scale horizontally. Bottleneck: slow LLM calls + rate limits → need a queue/concurrency limit at scale.
- **quan-quen-map**: **client-only, no backend** → private, static deploy; cross-device sync via GitHub Gist (trade-off: manual). At larger scale: move storage to a DB/object store.
- **totp-manager**: single user, local-only → no distribution needed; for multi-user you'd add auth + per-user data isolation.
- **outlook-mail-reader**: depends on external services (Graph/IMAP) → needs **retry + fallback** (done), and threading for bulk tasks.

## Self-practice (apply the 6-step framework)
- Design a backend for a multi-user 2FA manager (hint: from local totp-manager → add auth, per-user encrypted vaults).
- Design a Word-checking system at scale (hint: from word-checker CLI → API + a file-processing queue + result storage).

## Sample questions
- "How do you scale for many users?" → stateless + load balancer + cache + a queue for heavy work.
- "SQL or NoSQL?" → depends on data relations + consistency needs; state the trade-offs.
- "Handling a slow/rate-limited external service?" → retry with backoff + fallback + cache (e.g. fetchWithRetry in quan-quen-map).

> 💡 The most important thing is to **clarify + think out loud + name trade-offs**. The interviewer grades how you think, not a single "right" answer.
