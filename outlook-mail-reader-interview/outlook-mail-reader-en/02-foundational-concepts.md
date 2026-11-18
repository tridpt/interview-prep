# 2. Foundational Concepts (theory)

This file explains **every term** in the project. It's heavy on **OAuth/identity**, so this part matters a lot.

---

## 2.1. OAuth 2.0 — Delegated access (the project's core)

**The problem:** how can an app read your mail **without knowing your password**?

**OAuth 2.0** solves it: you log in directly with Microsoft (not through the app), and Microsoft grants the app a **token** (a ticket) to access resources within the **scope** you allow. The app never sees the password.

**OAuth concepts to remember:**
- **Access token**: a short-lived ticket (tens of minutes – 1 hour) to call the API. Must be renewed when it expires.
- **Refresh token**: a long-lived ticket to **exchange for a new access token** without logging in again. This is what the app stores (encrypted) for "log in once, use forever".
- **Scope**: the permission range. The project requests `Mail.ReadWrite` (read + mark-as-read) + `User.Read`. `offline_access` is the scope that **grants a refresh token**.
- **Client ID**: the app's identifier registered with Microsoft (Azure App Registration).
- **Authority**: the identity server, here `login.microsoftonline.com/common` (`common` = both personal outlook.com and organizational accounts).
- **Public client**: an app with no "client secret" (a desktop/native app running on the user's machine) → needs "Allow public client flows" enabled in Azure.

---

## 2.2. Device Code Flow — the project's login

OAuth has several "flows". The project uses **device code flow** because it fits apps without an embedded browser:

1. The app calls Microsoft → receives a **short code (user_code)** + a **link** (`verification_uri`, usually `microsoft.com/devicelogin`).
2. The app shows the code + link to the user.
3. The user opens the link in any browser, enters the code, and logs into the real Microsoft.
4. Meanwhile the app **waits in the background (polls)** until the user finishes → receives the token.

**In the project:** `begin_login()` creates the flow + runs a **background thread** (`acquire_token_by_device_flow` blocks until done); the frontend **polls** `/login/status/{job_id}` every 2.5s. A classic **long-running task + polling** pattern.

---

## 2.3. MSAL — Microsoft Authentication Library

**What is MSAL?** Microsoft's official library handling all the hard OAuth parts:
- `PublicClientApplication(client_id, authority, token_cache)` — the app object.
- `initiate_device_flow` / `acquire_token_by_device_flow` — device code flow.
- `acquire_token_silent(scopes, account)` — **gets a new access token without prompting the user** (auto-uses the refresh token in the cache). This is the "magic" that avoids re-login.
- `get_accounts()` / `remove_account()` — manage **multiple accounts in one cache**.
- `SerializableTokenCache` — a cache serializable to a string for save/load.

➡️ **Cool point:** MSAL manages refresh tokens for **multiple accounts** in one cache. The project only adds **encryption of the cache** before writing to disk.

---

## 2.4. Microsoft Graph API

**Graph API** is Microsoft 365's unified REST API for mail, calendar, contacts... Base URL: `https://graph.microsoft.com/v1.0`.

Endpoints used (via `requests` with header `Authorization: Bearer <access_token>`):
- `GET /me/messages` — mail list (with `$top`, `$skip`, `$orderby`, `$select`, `$filter`).
- `GET /me/mailFolders/{folder}/messages` — mail by folder.
- `GET /me/messages/{id}` — one message detail (body HTML/text).
- `GET /me/messages?$search="..."` — full-text search (needs header `ConsistencyLevel: eventual`).
- `PATCH /me/messages/{id}` `{isRead: true/false}` — mark read.
- `GET .../attachments` — attachments.
- `GET /me/mailFolders` — folder list + unread counts.

**OData query** ($top/$skip/$filter/$select/$orderby/$search): Graph's query syntax. E.g. `$filter=isRead eq false and hasAttachments eq true`.

---

## 2.5. IMAP + XOAUTH2 (the fallback mode)

Some accounts/refresh tokens work only over **IMAP** (an older mail-reading protocol), not Graph. The project has a fallback:
- Connect `imaplib.IMAP4_SSL("outlook.office365.com", 993)`.
- Authenticate via **XOAUTH2**: instead of a password, send the string `user=<email>\x01auth=Bearer <access_token>\x01\x01` (still an OAuth token, not a password).
- Read mail with IMAP commands (`SELECT`, `UID SEARCH`, `UID FETCH`), parse with `email`/`BytesParser`.

➡️ The project **implements both paths** (Graph and IMAP) for every operation (list/detail/search/mark-read/attachments) — chosen by the account's `mode`.

---

## 2.6. Token encryption with Fernet (cryptography)

**Why encrypt?** A refresh token is the "key" to the mail — storing it plaintext is dangerous if the file leaks. The project encrypts before writing to disk.

- **Fernet** (in the `cryptography` library): convenient symmetric encryption (AES-128-CBC + HMAC), simple `encrypt`/`decrypt`.
- **Encryption key** generated once (`Fernet.generate_key()`), stored at `storage/cache.key`, with **owner-only read permission** (`chmod 600` on POSIX; safely ignored on Windows).
- The MSAL token cache (`SerializableTokenCache.serialize()`) and the "imported accounts" file are both Fernet-encrypted before writing.
- On read: decrypt; if corrupt/key changed → skip, user logs in again (doesn't crash the app).

> 💡 **Question: "What's the point of encrypting with a key stored right next to the encrypted file?"** Honest answer: it does **not** stop an attacker who has full control of the machine (the key is right there). It stops **accidental exposure** (a stray backup, an accidental git push, someone skimming the file). Stronger protection needs the key from an OS keychain/env var — a **known limitation** worth naming.

---

## 2.7. Multithreading & background jobs (threading, Queue)

Importing many refresh tokens (each requires a network call to verify) would be slow done serially. The project uses:
- **`Queue`**: a thread-safe queue of lines to process.
- **Worker threads** (1-5, user-chosen): each thread takes work from the queue, processing in parallel.
- **`threading.Lock`**: protects shared job state (counting processed/added/failed) from race conditions.
- The job stores progress and a log; the frontend **polls** `/import-jobs/{id}` to show %; there's a **cancel** button (sets `cancel_requested`, workers stop themselves).

➡️ A classic **producer/consumer + job tracking** pattern, done by hand with the standard library.

---

## 2.8. DNS-over-HTTPS & MX (the email-check feature)

When "checking email live", the project doesn't send a test email — it looks up the domain's **MX (Mail eXchanger) records**:
- Calls **Cloudflare DNS-over-HTTPS** (`cloudflare-dns.com/dns-query?name=<domain>&type=MX`) — DNS over HTTPS (no DNS library needed).
- If the domain has MX → it **can likely receive mail**; guesses the provider (microsoft/google/yahoo...) from the MX hostnames.
- It **does not verify the specific mailbox** (with just an address you can't be sure the mailbox exists).
- Handles internationalized domains (IDN) via `encode("idna")`.

---

## 2.9. Safe mail-HTML rendering — iframe sandbox

HTML mail from strangers may contain **malicious scripts/trackers**. Rendering it directly into the page is an XSS risk. The project uses:
- **`<iframe sandbox>`**: embeds the mail HTML in an "isolated" iframe — `sandbox` (with no value) **disables everything**: no scripts, no forms, no escaping to the parent page.
- Content is set via `srcdoc`. Text mail is wrapped in `<pre>` after `escapeHtml`.

> 💡 A strong security point worth stating: "I render mail HTML inside a `sandbox` iframe for full isolation — scripts in the mail can't run, no trackers, no escaping into the main DOM."

---

## 2.10. Other terms

- **FastAPI / ASGI / Uvicorn / Pydantic**: as in other Python projects (async web, request validation via models).
- **CORS**: `allow_origins=["*"]` — lets any origin call the API (convenient for dev; needs tightening if public — see file 06).
- **Lazy import** `from engine import engine` **inside** each route: so `engine` (a singleton created at import time) isn't initialized too early and avoids import cycles.
- **Polling**: the client repeatedly asks the server for status (login, import job).
- **Singleton**: `engine = OutlookEngine()` at the end of the file — one shared instance for the whole app.
- **home_account_id**: the account identifier in MSAL; for imported accounts it's `imported:<short sha256>`.
