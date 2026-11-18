# 7. CS Fundamentals (Git, Docker, SOLID, complexity, crypto)

The "everyone should know" things that get sprinkled into interviews.

## 1. Complexity & data structures
- **Big-O**: O(1) < O(log n) < O(n) < O(n log n) < O(n²). Good sort = O(n log n).
- **Arrays** (O(1) access, O(n) mid-insert) · **hash map/set** (O(1) average lookup) · **stack/queue** (LIFO/FIFO) · **trees/graphs** (DFS/BFS traversal).
- Tip: "find/count/duplicate" → think **hashing**.

## 2. Git
- `add` → `commit` → `push`; branches: `branch`/`checkout`/`merge`.
- **merge vs rebase**: merge creates a merge commit (keeps true history); rebase rewrites history linearly (clean, but don't rebase shared branches).
- **Pull request / code review**: the collaboration workflow. `.gitignore` (excludes sensitive files — e.g. `.env`, `storage/vault.bin`).
- Conflict: when two sides edit the same region → resolve manually.

## 3. Docker (cap-cuu-101/agent-talk have a Dockerfile)
- **Image** (template) vs **Container** (running instance).
- `Dockerfile`: `FROM` (base image) → `COPY`/`RUN` (install) → `CMD` (run command).
- **Layer cache**: put dependency installation BEFORE copying code → faster rebuilds (code changes often, deps rarely).
- Pass config via environment variables (`-e LLM_API_KEY=...`).

## 4. SOLID & design principles
- **S**ingle Responsibility: one class/module, one job (word-checker: one module per check type).
- **O**pen/Closed: open to extend, closed to modify (add a check = add a function, don't edit the core).
- **L**iskov / **I**nterface Segregation / **D**ependency Inversion.
- **DRY** (Don't Repeat Yourself): a shared runner for CLI + web. **KISS**, **YAGNI**.
- **Separation of concerns**: reading data ≠ logic ≠ output (word-checker: document/checkers/report).

## 5. Process vs Thread vs Async
- **Process**: separate memory. **Thread**: shared memory within a process (beware race conditions → use a **Lock**, e.g. outlook's import job).
- **Async (event loop)**: single-threaded but yields while waiting on I/O — suits I/O-bound.
- Python GIL: threading suits I/O-bound; CPU-bound should use multiprocessing.

## 6. Basic cryptography (tied to totp-manager, quan-quen-map)
- **Hash** (SHA-256): one-way, irreversible; used for integrity.
- **HMAC**: a hash with a secret key for authentication (TOTP uses it).
- **Symmetric** (AES/Fernet): one key for both encrypt & decrypt.
- **Asymmetric** (RSA): a public/private key pair.
- **KDF** (scrypt/PBKDF2): password → key, deliberately CPU/RAM-heavy to resist brute-force. **Salt** defeats rainbow tables; **IV** makes the same data produce different ciphertext.
- **Encryption-at-rest**: data stored already encrypted (vault.bin).

## 7. Databases (basics)
- **SQL** (relational, ACID, joins) vs **NoSQL** (flexible, horizontal scaling).
- **Indexes** speed reads, at the cost of write speed + storage.
- **Transaction / ACID**: Atomicity, Consistency, Isolation, Durability.

## Sample questions
- "merge vs rebase?" · "Why install deps before copying code in a Dockerfile?" · "SOLID — the S?" · "Hash vs encryption?" · "What's a race condition, how to prevent?" (Lock) · "What do indexes trade off?"

> 💡 For each concept, have **an example from your own project** — turning dry theory into a real story is far more impressive.
