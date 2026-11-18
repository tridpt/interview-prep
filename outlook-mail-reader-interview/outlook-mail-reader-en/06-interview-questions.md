# 6. Interview Question Bank & Sample Answers

70+ questions, grouped, easy to hard. Each has a suggested answer. Rephrase in your own words.

> ⭐ = very commonly asked.

---

## A. Overview & motivation

**A1. ⭐ Briefly introduce your project.**
> Outlook Mail Reader merges multiple Outlook/Microsoft 365 inboxes into one place. Each account logs in via OAuth once (device code flow), then the app auto-uses a refresh token (stored encrypted) to fetch mail without re-login; no passwords stored. Backend FastAPI + MSAL + Microsoft Graph (with IMAP fallback): unified inbox, search, mark-as-read, attachments, multithreaded token import. Frontend vanilla JS, renders mail HTML in a sandboxed iframe.

**A2. Why build this?**
> I use several Outlook accounts and wanted them merged, plus I wanted to practice real OAuth/identity integration — something almost every product needs. I learned the OAuth flow, refresh tokens, and secure token storage.

**A3. Who are the users?**
> People with several Outlook/M365 accounts who want them in one place, privately (runs locally, tokens on their own machine).

**A4. ⭐ Is the project safe/legitimate?**
> Yes: it uses Microsoft-standard OAuth, stores no passwords, and each account still goes through the real Microsoft login once — so only for accounts you own/are authorized for. The requested scope is Mail.ReadWrite (read + mark-as-read, no send/delete). Tokens stored encrypted.

---

## B. OAuth & MSAL (the focus)

**B1. ⭐ Access token vs refresh token?**
> Access tokens are short-lived (tens of minutes) for API calls; when they expire you renew. Refresh tokens are long-lived, used to exchange for new access tokens without re-login — that's what the app stores (encrypted) for "log in once, use forever".

**B2. ⭐ How does device code flow work? Why use it?**
> The app asks Microsoft for a short code + link; the user opens the link, enters the code, logs in for real; the app polls in the background until done then receives the token. Used because the app has no web redirect/embedded browser — this flow is simple and safe for public clients.

**B3. ⭐ What does MSAL provide? Why not hand-roll OAuth?**
> MSAL (Microsoft's official library) handles device flow, auto-refresh, multi-account token cache, expiry. Hand-rolling is error-prone and less secure. I only wrapped cache encryption when writing to disk.

**B4. What is a scope? What does the project request?**
> The permission range. The project requests Mail.ReadWrite + User.Read, plus offline_access to be granted a refresh token. For read-only, switch to Mail.Read.

**B5. What does acquire_token_silent do?**
> Gets a new access token from the cached refresh token WITHOUT prompting the user. That's the mechanism that avoids re-login on later runs.

**B6. What does authority "common" mean?**
> An endpoint supporting both personal accounts (outlook.com/hotmail) and organizational accounts (Microsoft 365). For a single tenant you'd use the tenant id instead of "common".

**B7. What is a public client? Why "Allow public client flows"?**
> A native/desktop app on the user's machine can't keep a client secret safe → it's a public client. Device code flow needs this flag enabled in the Azure App Registration.

---

## C. Security & token storage

**C1. ⭐ How are tokens stored? Why encrypt?**
> MSAL serializes the cache (multi-account refresh tokens) → Fernet (AES) encrypt → write token_cache.bin. Because the refresh token is the key to the mail, never plaintext on disk.

**C2. ⭐ What's the point of encryption if the key is on the same machine?**
> Honestly: it doesn't stop an attacker who owns the machine (the key is right there). But it prevents accidental exposure — a stray backup, an accidental git push, someone skimming the file. Stronger protection needs an OS keychain/env var. I note this as a limitation.

**C3. Why no password storage despite the import line having a password column?**
> `_parse_import_line` splits 4 parts but assigns the password to a `_password` variable then drops it — using only email/refresh_token/client_id. Never stored or sent.

**C4. How is the key file protected?**
> `chmod 600` (owner read/write only) on POSIX; safely ignored on Windows. A corrupt/key-changed cache is skipped and the user logs in again.

**C5. What's the risk of rendering mail HTML? How handled?**
> Mail HTML may contain scripts/trackers → XSS. I set the body into a `sandbox` iframe (full isolation, no scripts, no parent-DOM escape); text is wrapped in an escaped `<pre>`.

---

## D. Microsoft Graph & IMAP

**D1. ⭐ What is the Graph API? Which endpoints?**
> Microsoft 365's unified REST API. I use /me/messages (list, with $top/$skip/$orderby/$select/$filter), /me/messages/{id} (detail), $search (search), PATCH isRead (mark), /attachments, /mailFolders.

**D2. ⭐ Why also IMAP? When is it used?**
> Some refresh tokens only work over IMAP (not Graph). On import, if Graph fails it falls back to IMAP (mode "imap"). Every mail operation has two branches by the account's mode.

**D3. How does IMAP authenticate with OAuth (XOAUTH2)?**
> Instead of a password, send `user=<email>\x01auth=Bearer <access_token>\x01\x01` via the AUTHENTICATE XOAUTH2 command — still an OAuth token, not a password.

**D4. $filter vs $search?**
> $filter filters by exact conditions (isRead eq false, hasAttachments eq true, date). $search does full-text, can't combine with $orderby and needs the ConsistencyLevel: eventual header.

**D5. Why filter sender client-side?**
> Graph $filter doesn't support fuzzy-matching the sender address, so after loading a mail page, the app filters by `sender` (matching name/address) on the client.

**D6. How is the IMAP mail id encoded?**
> `imap:<base64(folder)>:<uid>` — because an IMAP mail id is tied to a folder + UID; encoded for safe URL transport, decoded back when needed (validating the UID is numeric).

---

## E. Multithreading & jobs

**E1. ⭐ How does the multithreaded import job work?**
> Push every line into a Queue; start 1-5 worker threads taking work in parallel; each imports one line (network verification); a Lock protects processed/added/failed; the frontend polls progress %; a Cancel button sets cancel_requested.

**E2. What is a race condition? How do you prevent it?**
> Multiple threads updating shared job state (processed++, results.append) can conflict and corrupt counts. I use a threading.Lock around every read/write of shared state.

**E3. Why are workers daemon threads?**
> Daemon threads end when the main process exits → no hanging server. Jobs live in RAM, lost on restart (acceptable for short tasks).

**E4. How does the frontend know the job is done?**
> Poll GET /import-jobs/{id} every 1.5s until status is done/cancelled; then load all results (paginated 200/page) to show copy/retry of failed lines.

**E5. Login also runs in the background — same pattern?**
> Yes. begin_login runs a thread waiting on the device flow, and the frontend polls /login/status. Both are the long-running task + polling pattern.

---

## F. FastAPI & architecture

**F1. ⭐ Describe the backend architecture.**
> config (settings) ← store (token encryption) ← engine (logic: OAuth, Graph/IMAP, jobs) ← routes (HTTP) ← main (app + mount frontend). engine is a singleton with a Lock; state in RAM + encrypted files.

**F2. Why lazy-import engine in each route?**
> To avoid import cycles and avoid initializing OutlookEngine (file reads, MSAL setup) too early at module import. Engine is a singleton, so one shared instance.

**F3. Is the backend stateful?**
> Yes. Token cache + imported accounts in encrypted files; login/import jobs in RAM. Unlike client-only apps — because it must hold the OAuth session.

**F4. How is error handling done?**
> The engine raises RuntimeError with clear messages; routes catch and convert to HTTPException 400/404. Each account/import line is wrapped in its own try/except to isolate errors.

**F5. Is CORS allow_origins=* a problem?**
> Convenient for dev but too broad if public. This app is designed for local use. For public deployment, restrict origins + add authentication (see I3).

---

## G. Specific features

**G1. How is the unified inbox loaded?**
> get_unified_inbox loops each account, calls get_messages (Graph or IMAP), attaches messages + has_more; one account's error gets its own error field. The frontend paginates per account (skip = page × PAGE).

**G2. What does mark-as-read require?**
> Mail.ReadWrite. Graph: PATCH isRead. IMAP: STORE +FLAGS \Seen. If the token is Mail.Read-only → an error saying ReadWrite is needed.

**G3. ⭐ What does the live email check do? Is it certain?**
> Validate format + look up MX via Cloudflare DNS-over-HTTPS + guess the provider. It only says the domain CAN likely receive mail; it does NOT verify the specific mailbox exists (with just an address you can't be sure).

**G4. What does account health check verify?**
> IMAP: opens INBOX. Graph: reads 1 mail. Classifies errors: token_expired / missing_scope / graph_error / imap_error...

**G5. How is the unread-count badge fetched?**
> /unread-counts calls inbox_unread_count per account (Graph unreadItemCount or IMAP UNSEEN search); errors → null (the badge is secondary).

---

## H. Open-ended / thinking

**H1. ⭐ Hardest challenge?**
> (Hint) Abstracting two backends (Graph/IMAP) for every operation while keeping it clean; or making the multithreaded import job safe (Lock, cancel, progress) plus scope probing on import. Problem → approach → result.

**H2. ⭐ What would you improve with more time?**
> Source the encryption key from an OS keychain/env var; add authentication to the local API + tighten CORS; persist jobs to durable storage (survive restart); add compose/send, new-mail notifications, read-aloud TTS (the project's roadmap); split the 1,524-line engine.py into smaller modules.

**H3. ⭐ Where does the app break with many accounts/mails?**
> get_unified_inbox calls accounts serially → many accounts get slow (could parallelize); Graph/IMAP have rate limits; IMAP opens/closes a connection per operation (could pool). Expired tokens → health check + change token.

**H4. The technical decision you're most proud of?**
> Proper OAuth/MSAL integration + encrypted token storage + safe HTML rendering. It's real identity backend work, not a CRUD demo.

**H5. The biggest trade-off?**
> The refresh-token import feature is powerful (batch account management) but requires clearly stating the ethical/legal constraint "only accounts you own". Local + open CORS → simple but needs tightening if public.

---

## I. "Specific code" questions

**I1. What does _token_for() do?**
> The token entry point for every Graph call: imported graph → _refresh_imported_access_token (redeem refresh token); MSAL → acquire_token_silent. Imported IMAP → raises "no Graph token".

**I2. How does _try_graph_import probe scopes?**
> Tries scope sets in order (Mail.Read/Mail.ReadWrite/...); for each, redeems a token then GET /me/messages?$top=1; the first 200 wins (with can_write); on exhaustion it raises a combined error.

**I3. When does save_cache write?**
> Only when cache.has_state_changed = True (MSAL signals the cache changed) → avoids redundant writes.

**I4. Does _redeem_refresh_token update the token?**
> Yes — refresh tokens can rotate: if the response returns a new refresh_token different from the old one, it's saved (with a Lock) for next time.

**I5. Why does opening mail auto mark-as-read but stay silent on error?**
> openMail calls setRead(true, refresh=false); if the token is Mail.Read-only it errors "Mail.Read" → caught and ignored (no annoying alert), since marking is secondary.

**I6. What is FOLDER_MAP for?**
> Maps friendly names (inbox/sent/drafts/junk/deleted/archive) → Graph well-known folders (inbox/sentitems/drafts/junkemail/deleteditems/archive).

---

## 🎤 The 5 most common — rehearse first
1. A1 — Introduce the project.
2. B1/B2 — Access vs refresh token & device code flow.
3. B3 — MSAL & why not hand-roll OAuth.
4. C1/C2 — Encrypted token storage & its limit.
5. D2 — Why both Graph and IMAP.

## ❓ Questions YOU should ask the interviewer
- Does the product integrate OAuth/SSO (Microsoft/Google), and with what libraries?
- How does the team store and rotate tokens/secrets (keychain/vault)?
- Do you use background jobs/queues for long tasks, with what tech (Celery/RQ...)?
