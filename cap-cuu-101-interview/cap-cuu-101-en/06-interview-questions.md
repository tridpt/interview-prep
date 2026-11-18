# 6. Interview Question Bank & Sample Answers

60+ questions grouped by topic, easy to hard. Each has a **suggested answer**. Read them, then rephrase in your own words.

> ⭐ = very commonly asked. Prioritize these.

---

## A. Overview & motivation (warm-up)

**A1. ⭐ Briefly introduce your project.**
> Cấp Cứu 101 is a PWA web app that teaches life-saving first aid through funny Dumb-Ways-to-Die-style mini-games, plus a Panic Mode that reads each step aloud for real emergencies. It's all plain HTML/CSS/JS — no framework, no build, no libraries — and runs 100% offline via a service worker. It uses many Web APIs (Web Audio, Web Speech, Vibration, DeviceOrientation, Web Share), is bilingual VI/EN, and stores data locally.

**A2. Why did you build this?**
> I wanted something genuinely useful (helping people remember life-saving skills) that was also a technical challenge — especially the offline-first requirement for an emergency app, and doing everything in vanilla JS with no dependencies.

**A3. Who are the users?**
> Everyday people who want to learn first aid gently, older adults (helped by large text/high contrast), and classrooms/medical settings (serious mode).

**A4. ⭐ Why "Don't Die Dumb" / why humor for a medical app?**
> Inspired by Dumb Ways to Die — using humor and satire so the knowledge sticks naturally, subconsciously. But I'm aware of the risk, so there's a **Serious Mode** that removes the dark humor for medical settings, plus clear disclaimers: "not a replacement for formal training, call 115 for real."

---

## B. Architecture & design

**B1. ⭐ Describe the overall architecture.**
> Fully client-side, no backend. `index.html` holds 10 screens as sections; `sound.js` exposes window.Sfx (Web Audio sound); `app.js` is the brain (i18n, router, data, game engine, Panic, tools) wrapped in an IIFE; `sw.js` is the service worker for offline. All user data is in localStorage.

**B2. ⭐ Why no backend?**
> Not needed — the content is static and all logic runs on the client. No backend means: works offline, total privacy (personal medical data is never sent anywhere), simple deployment (static GitHub Pages), no server to crash or maintain.

**B3. ⭐ Why vanilla JS, no framework?**
> The biggest requirement was offline-first and absolute lightness. Every dependency is a risk (size, supply-chain security, build complexity). Vanilla gives full control and a tiny app shell to cache. The cost is writing my own router/i18n/state — which is also where I learned the most.

**B4. Downside of putting all logic in one large app.js?**
> A ~2,980-line file is harder to maintain as it grows. I've noted this as a limitation; the improvement path is splitting into ES modules at larger scale. I keep one file for now to avoid a build step.

**B5. How does your router work?**
> The `go(id)` function: stop the old screen's activity (game/panic/metronome to avoid leaks), switch the `.active` class among sections, update `aria-hidden`, then call the matching render. Simple client-side routing with no page reload.

**B6. Where is app state stored?**
> All in localStorage: stars, stats, sound settings, accessibility prefs, language, voice, medical card. Each type has its own `capcuu101_`-prefixed key.

---

## C. PWA & Offline (the focus)

**C1. ⭐ What is a PWA? What does it require?**
> A web app installable to the home screen and running offline like a native app. Requires: HTTPS (or localhost), a Web App Manifest, and a Service Worker.

**C2. ⭐ How does the service worker work in the project?**
> On install: precache the whole app shell into the Cache API. On activate: delete old caches not matching the current version. On fetch: intercept requests — page navigation uses network-first, assets use stale-while-revalidate.

**C3. ⭐ Explain the two caching strategies and why they differ.**
> Network-first for HTML pages: try the network first (to get the latest), fall back to cache offline → never stuck on an old version when online. Stale-while-revalidate for assets: return cache immediately (fast), update in the background for next time → assets rarely change so speed is prioritized.

**C4. ⭐ How do users get an update?**
> Bump `CACHE_VERSION` on every asset change. The new SW, on activate, deletes old caches; `skipWaiting()` + `clients.claim()` apply it immediately without closing the app.

**C5. Why doesn't the service worker run with `file://`?**
> For security — SWs only run on `https://` or `localhost`. Opening a file directly won't register the SW, so you must serve over HTTP (e.g. `python -m http.server`).

**C6. What is `Promise.allSettled` for during precache?**
> So one failed file (e.g. missing) doesn't break the entire SW install — the others still get cached. Safer than `Promise.all` (which fails if one fails).

**C7. How important is offline for this project?**
> Vital — it's an emergency app, and emergencies may lack network. Panic Mode must work offline. That's why I prioritized offline-first from the start.

---

## D. Web APIs

**D1. ⭐ Why synthesize audio with Web Audio instead of using files?**
> To keep the app shell ultra-light and perfectly offline — no media to cache. I combine oscillators (waves) + gain (volume envelopes) to create every effect and the 8-bit music from scratch.

**D2. ⭐ What is the autoplay policy? How do you handle it?**
> Browsers block audio before user interaction. I have an `unlock()` that touches/`resume()`s the AudioContext on the first tap to unlock audio.

**D3. ⭐ What problems does TTS (Web Speech) have, and how do you solve them?**
> getVoices() is async (must listen for onvoiceschanged); voices depend on the OS — missing a vi-VN voice means an English voice mangles Vietnamese. I handle it with a voice picker, a warning when the Vietnamese voice is missing, and an option to turn off read-aloud.

**D4. What's the "one one five" → "115" trick for?**
> So TTS reads "one-one-five" instead of "one hundred fifteen". The source data writes "một một năm"; when displayed, the code swaps it back to "115". Similarly "năm centimet" → "5cm".

**D5. Where is DeviceOrientation used? What's the fallback?**
> The Heimlich "tilt to thrust" game uses the gyroscope. iOS requires permission; devices without a sensor fall back to press-hold-release.

**D6. What's the limitation of Notifications here?**
> It only reminds when the app is reopened (no background push). Background push even when the app is closed needs web-push infrastructure (server + VAPID) — not built, since there's no backend.

**D7. How does the Web Share API work? Is there a fallback?**
> It invokes the native share dialog (`navigator.share`). I draw the result card with Canvas then share it; unsupported devices fall back to copying a link to the clipboard.

---

## E. i18n

**E1. ⭐ What's your multilingual mechanism?**
> A LANG variable from localStorage; a t() function returns a string per language if the data is a {vi,en} object. Static text uses data-i18n + applyLang() setting textContent; dynamic content is stored bilingually in the structures; long prose uses .only-vi/.only-en classes.

**E2. Why roll your own i18n instead of a library?**
> The scale is small and needs are simple; rolling my own avoids adding a dependency (matching the project's no-deps philosophy). A library would be overkill and bloat the app shell.

**E3. What's needed to add a new string?**
> Add the same key to BOTH vi and en in UI; HTML uses data-i18n, JS uses t(UI.key). Dynamic content must update both versions (e.g. QUIZZES and QUIZZES_EN).

---

## F. Game engine & data

**F1. ⭐ How many game types? How do they reuse a common shell?**
> 5 types: rhythm, thrust (tilt), hose, press (bleeding control), quiz. 17 levels use only these 5 mechanics. They share 3 DOM elements and a gameState variable; runGame() dispatches by `type`.

**F2. ⭐ What is `gameState.cleanup` and why does it matter?**
> Each game, on init, sets a cleanup function (remove listeners, clear intervals, stop sensors). stopGame() calls it on leaving. No framework handles this, so I must tear down to prevent resource leaks — e.g. a metronome running forever or a sensor still listening.

**F3. The data model for a level?**
> LEVELS[] has {id, emoji, type, char, title{}, desc{}, lesson{}}. Quizzes live in QUIZZES[id] = an array of questions, each options:[{t,ok}] with exactly one correct. Panic lives in PANIC[id] = {title, metronome?, steps[]}.

**F4. How is Survival mode different from a normal game?**
> Continuous random questions from all situations, each with a countdown (~9s via requestAnimationFrame), one wrong/timeout ends it, saves best.survival.

**F5. Why do the Handbook and Panic share data?**
> The DRY principle and medical content safety: one data source, fix one place is correct everywhere, never inconsistent between the two screens.

---

## G. Security & privacy

**G1. ⭐ Is user data safe?**
> All data (stars, stats, medical card) is in localStorage on the device, never sent anywhere. No backend means no server-side leak risk.

**G2. ⭐ Where is XSS prevented?**
> The medical card takes user input → rendered via `textContent` (escaped) instead of innerHTML so injected code can't run. Phone numbers are filtered to `[\d+]` before building `tel:`.

**G3. Why is "no external libraries" also a security point?**
> No embedded library/CDN means no client-side supply-chain attack surface — no worry about a dependency being injected with malicious code.

**G4. What's the medical card's limitation?**
> It's inside the app — if the screen is locked, rescuers can't open it. It should be used alongside the OS Medical ID. I note this as a limitation.

---

## H. Accessibility & medical content

**H1. ⭐ What do you do for accessibility?**
> High contrast, large text, reduced motion (respecting prefers-reduced-motion), serious mode; ARIA on icon buttons, aria-live reading Panic steps, aria-hidden on hidden screens, clear focus, the COMPRESS button playable with Space/Enter, quizzes as buttons.

**H2. Are the touch games (drag/tilt) keyboard-accessible?**
> Hard — I acknowledge this. But the quiz part and Panic Mode are fully accessible learning channels for everyone, including keyboard/screen-reader users.

**H3. ⭐ What worries you most about a medical app? How do you handle it?**
> Wrong content is the biggest risk. I cross-referenced AHA/Mayo Clinic/NHS/WHO/Red Cross, have a MEDICAL_REVIEW.md checklist for medical staff, clear disclaimers in many places, and a single data source so content never diverges.

**H4. What problem does Serious Mode solve?**
> The "logged out of life" humor isn't appropriate for classrooms/medical settings. Serious mode drops the dark humor and uses neutral wording ("Completed correctly" / "Not yet — try again").

---

## I. Testing & operations

**I1. ⭐ How is the project tested?**
> Honestly: no automated tests yet — a known limitation. Currently it's manual testing per a checklist (HUONG_DAN_TEST.md) + syntax checks with `node --check` for JS and `python -m json.tool` to validate the manifest.

**I2. If you added tests, what first?**
> Data integrity checks — the most dangerous thing for a medical app: every quiz has exactly one correct answer, every quiz has both VI+EN, every panic-card has matching data. This class of content bug is easy to slip in and dangerous.

**I3. How do you deploy?**
> GitHub Pages — it's static, so deploy = push files. After changing assets, remember to bump CACHE_VERSION so clients get the new build.

**I4. What if you get "stuck" on an old build during dev due to the service worker?**
> That's stale-while-revalidate. The fix: Ctrl+F5 or Unregister the service worker in DevTools, and remember to bump CACHE_VERSION.

---

## J. Open-ended / thinking

**J1. ⭐ The hardest challenge in the project?**
> (Hint) Handling cross-platform TTS (async getVoices, missing Vietnamese voice, reading numbers correctly); or ensuring offline truly works with cache versioning. Tell the problem → how you solved it → the result.

**J2. ⭐ What would you improve with more time?**
> Add automated tests (especially medical data integrity); split app.js into modules; real web push (server + VAPID) for background reminders; have Vietnamese medical staff + native speakers review the content/translation; full WCAG evaluation with a screen reader.

**J3. Where does the app "break" on weak/feature-poor devices?**
> No vi-VN voice → TTS is wrong (warned + can disable); no gyroscope → the tilt game falls back to tap; no Web Share → copy a link; vibration is mostly Android. I designed fallbacks for most of these.

**J4. Why use Python here if it's a pure JS app?**
> Python (Pillow) is only used during dev to generate the icon/screenshot/Open Graph images (gen_*.py), run once, not part of the app runtime.

**J5. The technical decision you're most proud of?**
> A complete offline-first architecture in vanilla — service worker + synthesized audio (no media files) → a tiny app shell that runs 100% offline for an emergency app. Every piece serves one clear goal.

**J6. The project's biggest trade-off?**
> No-framework/no-build (light, great offline, no deps) at the cost of writing a lot yourself and a large app.js that's hard to scale. And the humor style (memorable, shareable) at the cost of needing a serious mode + disclaimers for medical settings.

---

## K. "Specific code" questions (you may be quizzed)

**K1. Does the load order of sound.js and app.js matter?**
> Yes — sound.js loads first to expose window.Sfx, which app.js uses. Reversing it would throw "Sfx is undefined".

**K2. Why wrap app.js in an IIFE?**
> To create a private scope (variables don't leak to global, avoiding clashes) and enable "use strict". With no module system (no build), an IIFE is the simple way to isolate code.

**K3. What are `?panic=1` and `?tools=1` for?**
> Query parameters for the 2 PWA shortcuts — opening the app straight into Panic Mode / Emergency Tools. Important for emergencies, no multi-step navigation.

**K4. When switching language mid-session, does the open screen update?**
> Yes — setLang() saves to localStorage, calls applyLang(), then re-renders the current screen so dynamic content (quiz/panic...) updates too.

**K5. Why is the result overlay attached to #screen-game, not #game-stage?**
> So it covers the full screen and scrolls; attaching to the stage would confine it to the game frame.

---

## 🎤 The 5 most common questions — rehearse these first
1. A1 — Introduce the project.
2. C2/C3 — Service worker & the two caching strategies.
3. B3 — Why vanilla JS, no framework.
4. D3 — Handling cross-platform TTS.
5. H3 — What worries you about a medical app, and how you handle it.

## ❓ Questions YOU should ask the interviewer
- Does the team build PWAs/offline-first, and what caching strategy do they use?
- What's the QA process and content-quality assurance (especially in sensitive domains)?
- How much does the team prioritize accessibility in the product?
