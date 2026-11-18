# 7. Cheat Sheet — Quick Review Before Go-Time

A one-pager. Skim for 5-10 minutes right before the interview.

---

## 🎯 30-second pitch (memorize)
> Cấp Cứu 101: a **PWA** web app teaching first aid through funny Dumb-Ways-to-Die-style mini-games, plus a **Panic Mode** that reads each step aloud for real emergencies. 100% **plain HTML/CSS/JS** — no framework, no build, no libraries — **runs 100% offline** via a service worker. Uses many Web APIs (Web Audio, Web Speech, Vibration, DeviceOrientation, Web Share), **bilingual VI/EN**, data stored locally (localStorage).

## 🧱 Tech stack
- **Core**: Vanilla HTML/CSS/JS (ES6+), no build step.
- **PWA**: Web App Manifest + Service Worker + Cache API.
- **Web APIs**: Web Audio (synth sound), Web Speech (TTS), Vibration, DeviceOrientation, Notifications, Web Share, Canvas, localStorage.
- **Dev tools**: Python + Pillow (gen_*.py image generation — not runtime).
- **Deploy**: GitHub Pages (static).

## 🗂️ Files
| File | Role |
|---|---|
| `index.html` | 10 screens (section.screen) + overlays, PWA/OG meta |
| `sound.js` | window.Sfx — Web Audio synth (loaded BEFORE app.js) |
| `app.js` | The brain (IIFE ~2,980 lines): i18n, router, data, engine, Panic, tools |
| `style.css` | Theme, animation, a11y |
| `sw.js` | Service worker — offline |
| `manifest.json` | PWA declaration + 2 shortcuts |

## 📐 Architecture
Client-side only, **no backend**. All logic + data + state in the browser. localStorage holds stars/stats/prefs/lang/voice/medical card.

## 🌊 Service Worker (know it cold)
- **install** → precache app shell (Promise.allSettled) → skipWaiting.
- **activate** → delete old caches ≠ CACHE_VERSION → clients.claim.
- **fetch** → navigate: **network-first**; assets: **stale-while-revalidate**.
- Change assets → **bump CACHE_VERSION** (e.g. v17→v18).
- SW only runs over `http(s)://`, not `file://`.

## 🔊 Web Audio (synthesized)
Oscillator (wave) + Gain (envelope) → tick/ting/lose/win/music. No media files → light app, great offline. **Autoplay policy**: `unlock()` audio on the first user gesture.

## 🗣️ TTS (Panic Mode)
getVoices() is **async** (listen for onvoiceschanged); missing vi-VN voice → English mangles Vietnamese → **warning + voice picker + off option**. The "one one five" trick → reads "115", displays "115".

## 🎮 Game engine
5 types: rhythm / thrust / hose / press / quiz (17 levels). Each game sets **`gameState.cleanup`**; `stopGame()` calls it on leaving → prevents resource leaks. Survival: continuous questions + countdown (requestAnimationFrame).

## 🌐 i18n
LANG (localStorage) + `t()`; static text `data-i18n` + applyLang(); dynamic content `{vi,en}` / QUIZZES_EN, PANIC_EN; long prose `.only-vi/.only-en`.

## 🧩 Content safety (medical app)
**Single data source**: Handbook + Panic share PANIC/PANIC_EN (DRY). Cross-referenced AHA/Mayo/NHS/WHO. `MEDICAL_REVIEW.md` for medical staff. Disclaimer "not a replacement, call 115". **Serious mode** drops dark humor.

## 🔐 Security & privacy (3 points)
1. All data in localStorage, never sent anywhere.
2. Medical card: escaped via `textContent` (anti-XSS); `tel:` filtered to `[\d+]`.
3. No external libraries → no supply-chain attack surface.

## ♿ Accessibility
High contrast, large text, reduced motion (prefers-reduced-motion), serious mode; ARIA (label/live/hidden), clear focus, COMPRESS via Space/Enter, quizzes as buttons.

## ⚖️ Main trade-offs
No-framework/no-build → light, great offline, no-deps; at the cost of writing a lot yourself + a large app.js hard to scale. Humor → memorable/shareable; at the cost of needing a serious mode + disclaimers.

## 🚀 Future improvements
Automated tests (especially medical data integrity); split into modules; real web push (server+VAPID); native medical/translation review; full WCAG evaluation with a screen reader.

## ⚠️ Known limitations (raise proactively)
No automated tests yet; TTS depends on system voices; sensors/vibration/tel mostly on phones; the medical card is stuck behind a locked screen; app.js is a large file.

## 💬 Key terms
- **PWA / Service Worker / Cache API / app shell**
- **network-first / stale-while-revalidate / cache versioning**
- **Web Audio / oscillator / autoplay policy**
- **Web Speech (TTS) / async getVoices**
- **Vibration / DeviceOrientation / Web Share / Canvas**
- **i18n / IIFE / client-side routing**
- **localStorage / XSS / DRY / prefers-reduced-motion / ARIA**

---

### ✅ Pre-interview mental checklist
- [ ] Pitch the project in 30 seconds.
- [ ] Explain the service worker lifecycle + two caching strategies.
- [ ] The reason for choosing vanilla JS (offline-first, no-deps).
- [ ] How you handle cross-platform TTS.
- [ ] Articulate content-safety thinking for a medical app.
- [ ] Name 2 limitations + fixes.
- [ ] Prepared 2 questions to ask the interviewer.

**Breathe. You understand this project. Tell your story with confidence. 🍀**
