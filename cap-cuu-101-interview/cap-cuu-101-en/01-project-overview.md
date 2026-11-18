# 1. Project Overview — Cấp Cứu 101

## 1.1. What is Cấp Cứu 101?

Cấp Cứu 101 (full name: *Don't Die Dumb*) is a **web app that teaches first aid** through funny mini-games, inspired by *Dumb Ways to Die*. The core idea: **learn life-saving skills by playing**, so the knowledge sticks "subconsciously", and when a real emergency hits you have a tool to use right away.

Picture this: you play a "compress the chest of Fat Bear" mini-game to the correct rhythm of 100-120/min; get it wrong and your character "logs out of life" hilariously. But when a real incident happens, you open **Panic Mode**, pick the situation, and the app **reads each step aloud, calmly** for you to follow — with a metronome to keep CPR on beat.

### Core points to remember
- It's a **PWA (Progressive Web App)**: installable to the home screen like a real app, and **runs 100% offline**.
- **No backend, no framework, no build, no external libraries** — 100% plain HTML/CSS/JS.
- Two "halves": the **entertainment half** (learning mini-games) and the **real-tool half** (Panic Mode, CPR metronome, emergency numbers, medical card).
- **Bilingual VI/EN**, data stored locally (localStorage), never sent anywhere.

## 1.2. What problem does it solve?

1. **First-aid knowledge is dry and hard to remember** → turn it into a fun game so the brain retains it naturally.
2. **People panic and forget in an emergency** → Panic Mode reads each step aloud, with a "call 115" button always visible.
3. **No network exactly when you need it** → the offline PWA works even without Internet.
4. **Demonstrating technical skill**: mastering a stack of browser Web APIs without any library, offline-first architecture, i18n, accessibility — this is what scores points in an interview.

> 💡 **Interview tip**: when asked "what is this for", emphasize the **intersection of social value and technical difficulty**: "I wanted something genuinely useful (saving lives) that was also a technical challenge — especially the **offline-first** requirement for an emergency app, and doing everything in vanilla JS with no dependencies."

## 1.3. Main features

| Group | Features |
|-------|----------|
| **Learn by playing** | 17 mini-games (5 types: rhythm/thrust/hose/press/quiz), up to 3 stars per level, a real-life lesson after each |
| **Panic Mode** | 17 situations, read-aloud steps (TTS), a CPR metronome, a call-115 button |
| **Emergency tools** | CPR metronome (110/min), one-tap emergency calls (115/113/114/111), a personal medical card |
| **Quick handbook** | 17 situations as an accordion, for quick skimming |
| **Survival mode** | Continuous quiz with a countdown timer, saves a high score |
| **Achievements & retention** | Stars, 9 badges, a daily streak, study reminders |
| **Sound & vibration** | Synthesized with Web Audio (no media files), vibration on beat |
| **Sharing** | Draw an image card with Canvas + Web Share API, Open Graph for link previews |
| **Multilingual** | VI/EN across the whole UI + content + voice |
| **Accessibility** | High contrast, large text, reduced motion, serious mode, ARIA |
| **PWA & offline** | Install to home screen, 2 shortcuts, runs 100% offline |

## 1.4. Tech stack

**Core:**
- **HTML5 + CSS3 + plain JavaScript (vanilla, ES6+)** — no React/Vue, no jQuery, no libraries.
- **No build step**: open `index.html` and it runs (recommended over HTTP so PWA/voice/sensors work properly).

**Browser Web APIs used:**
- **Service Worker + Cache API** — caching for offline.
- **Web App Manifest** — declares the PWA (install to home screen, icons, shortcuts).
- **Web Audio API** — synthesizes all sound (no media files).
- **Web Speech API (SpeechSynthesis)** — reads Panic Mode steps aloud (TTS).
- **Vibration API** — vibrates to the CPR beat.
- **DeviceOrientation API** — tilt sensor for the Heimlich game.
- **Notifications API** — study reminders.
- **Web Share API + Canvas** — shares a result image card.
- **localStorage** — stores stars, stats, settings, medical card.

**Dev-only tooling (not runtime):**
- **Python + Pillow** — the scripts `gen_icons.py`, `gen_screens.py`, `gen_og.py`, `gen_gif.py` generate the icon/screenshot/Open Graph images (run once, not part of the app).

**Deployment:** GitHub Pages (static web, deploy = push files).

> 💡 **A trap question: "Why no framework?"** Good answer: "The biggest requirement was **offline-first and absolute lightness** for an emergency app — every KB and every dependency is a risk. Vanilla JS gives full control, **no supply-chain attack surface** from external libraries, no complex build, and a tiny app shell to cache offline. The cost is writing my own router, i18n, and state — which is also where I learned the most."

## 1.5. How to run (demo flow)

```bash
# Nothing to install. Run over HTTP so PWA/voice/sensors work:
cd cap-cuu-101
python -m http.server 8000
# Open http://localhost:8000
```
> The service worker **only runs over `http(s)://`** (localhost or GitHub Pages), not via `file://`.

Syntax checking (since there are no automated tests):
```bash
node --check app.js
node --check sound.js
node --check sw.js
python -m json.tool manifest.json   # validate JSON
```

## 1.6. Folder structure

```
cap-cuu-101/
├─ index.html   # 10 screens (section.screen) + splash/tutorial/settings overlays
├─ style.css    # dark cartoon theme (~590 lines)
├─ sound.js     # window.Sfx — Web Audio synth + sound settings (~150 lines)
├─ app.js       # all logic: i18n, router, content data, game engine, Panic, tools (~2,980 lines)
├─ sw.js        # service worker: precache app shell → offline (~80 lines)
├─ manifest.json# PWA declaration
├─ icon.svg, icons/, screenshots/   # image assets
├─ gen_*.py     # Python (Pillow) image scripts — dev only
└─ *.md         # README, FEATURES, DOCUMENTATION, CONTRIBUTING, MEDICAL_REVIEW...
```

Scale: ~4,200 lines total (app.js ~2,980, style.css ~590, sound.js ~150, index.html ~390, sw.js ~80).

**Script load order:** `index.html` loads `sound.js` **first** (exposes `window.Sfx`), then `app.js` (wrapped in an IIFE).

> ⚠️ **An important disclaimer (worth raising proactively):** this is an entertainment + memory-aid product, **not a replacement** for formal first-aid training or medical professionals. For real emergencies: **call 115 first**. The project even has a `MEDICAL_REVIEW.md` file as a checklist for medical staff to review the content — showing a sense of responsibility for a medical app.
