# 5. Deep Analysis of the "Point-Scoring" Features

Each feature: the idea, how it's done, "why it's cool".

---

## 5.1. OAuth 2.0 + MSAL: log in once, auto-refresh ⭐⭐ (the most important)

**Idea:** read mail without a password; log in once, use forever.

**How:** device code flow via MSAL → receive a refresh token → store it in the (encrypted) token cache. Later, `acquire_token_silent()` auto-exchanges the refresh token for a new access token. MSAL manages **multiple accounts in one cache**.

**Why it's cool:** this is **core backend knowledge** — nearly every enterprise product needs "Sign in with Microsoft/Google". Fluently explaining access vs refresh tokens, scopes, device code flow, and **silent token acquisition** is a huge plus.

**Possible follow-ups:**
- *"Access vs refresh token?"* → access is short-lived for API calls; refresh is long-lived to get new access tokens without re-login.
- *"Why device code flow?"* → for apps without an embedded browser/redirect: show a code + link, the user enters it in another browser.

---

## 5.2. Secure token storage (Fernet encryption) ⭐

**Idea:** a refresh token is the key to the mail — don't keep it plaintext.

**How:** serialize the MSAL cache → `Fernet.encrypt` (AES) → write `token_cache.bin`; key at `cache.key` (`chmod 600`); write only when `has_state_changed`. Imported accounts encrypted separately.

**Why it's cool:** shows security awareness. **Important — state the limitation:** the key is on the same machine, so it doesn't stop an attacker who owns the machine; it prevents accidental exposure (stray backup/git push). Stronger protection needs an OS keychain. Naming limitations shows maturity.

---

## 5.3. Two-backend abstraction: Graph + IMAP ⭐

**Idea:** some refresh tokens only work over IMAP, not Graph → the app must support both, transparently to the frontend.

**How:** each account has a `mode` (graph/imap). **Every** operation (`get_messages`, detail, search, mark_read, attachments, folders, unread) checks the mode then branches: Graph (REST + Bearer token) or IMAP (XOAUTH2 + SELECT/SEARCH/FETCH + `email` parse).

**Why it's cool:** a genuine **adapter/strategy layer** — one interface, two implementations. It also shows understanding of both modern REST APIs and the traditional IMAP protocol (XOAUTH2, UID, FLAGS `\Seen`).

---

## 5.4. Scope probing on token import ⭐

**Idea:** a refresh token may have `Mail.Read` or `Mail.ReadWrite` — the app must learn which to display correctly and block writes if missing.

**How:** `_try_graph_import` tries scope sets one by one (`_graph_import_scope_options`), calling `GET /me/messages?$top=1` for each; the first that returns 200 is used, recording `graph_can_write`. When `mark_read` runs on a Mail.Read-only token → a clear "needs Mail.ReadWrite" error.

**Why it's cool:** handles the real-world "we don't know the token's permission upfront" — smart and defensive.

---

## 5.5. Multithreaded import job with progress & cancel ⭐

**Idea:** import refresh tokens in bulk (each needs a network verification) quickly, observably, and cancelably.

**How:** a `Queue` holds the work; 1-5 **worker threads** take work in parallel; a `threading.Lock` protects the processed/added/failed counters; the frontend polls progress %, paginated log; a Cancel button sets `cancel_requested` so workers stop; the job self-finalizes as done/cancelled. With **copy/retry of failed lines**.

**Why it's cool:** a classic **producer/consumer + job tracking** pattern, done by hand with the stdlib — shows understanding of **concurrency, race conditions, locks**.

---

## 5.6. Safe mail-HTML rendering via iframe sandbox ⭐

**Idea:** mail HTML from strangers may carry malicious scripts/trackers.

**How:** set the body into a `<iframe sandbox>` via `srcdoc` → full isolation (no scripts, no escaping to the parent DOM). Text mail uses an escaped `<pre>`.

**Why it's cool:** a clear security decision, best practice for displaying untrusted content.

---

## 5.7. Per-account error isolation ⭐

**Idea:** one account with an expired token / network error shouldn't take down the whole unified inbox.

**How:** `get_unified_inbox`/`search_messages` wrap **each account** in try/except, attaching an `error` to just that account while others return mail normally.

**Why it's cool:** **resilience** and UX thinking — local failure, not global failure.

---

## 5.8. Email check via DNS/MX (DNS-over-HTTPS)

**Idea:** check whether an address can receive mail **without sending a test email**.

**How:** validate format → look up MX via **Cloudflare DoH** → MX present = the domain can receive mail; guess the provider from MX names; handle IDN. Explicitly notes it **doesn't verify the specific mailbox**.

**Why it's cool:** uses DNS-over-HTTPS for MX lookup with no DNS library; honest about its limit (domain only, not mailbox).

---

## 5.9. "Feature ↔ technical concept" table (memorize)

| Feature | Concept you demonstrate |
|---------|-------------------------|
| OAuth + MSAL | OAuth 2.0, access/refresh tokens, scope, device code flow, silent refresh |
| Encrypted tokens | Fernet/AES, key management, security limitations |
| Graph + IMAP | adapter/strategy, REST + OData, IMAP XOAUTH2 |
| Scope probing | handling unknown permissions, defense |
| Multithreaded import job | threading, Queue, Lock, race conditions, job tracking |
| iframe sandbox | XSS security for untrusted content |
| Error isolation | resilience, per-unit try/except |
| Email check DNS/MX | DNS-over-HTTPS, MX records, IDN |

> 💡 **Storytelling strategy:** if asked "what are you most proud of?", pick **OAuth/MSAL** (core, most valuable identity knowledge) or the **Graph+IMAP abstraction** (adapter design). Both show you can do complex real-world integration.
