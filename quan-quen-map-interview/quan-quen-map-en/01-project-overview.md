# 1. Project Overview — Quán quen (Taste Map)

## 1.1. What is Quán quen?

Quán quen ("favorite spots") is a **map-based web app** for saving your favorite food/cafe spots and managing them **by personal taste**. Unlike Google Maps (generic community ratings), Quán quen lets you record **your own** impressions: this place is *tasty*, that one is *quiet*, another *has power outlets, good for working* or *good for a date*. The app then uses that data to suggest "where to go today", give directions, plan itineraries, and discover new places around you.

### Core points to remember
- **A personalized map**: rate 5 personal criteria (tasty/quiet/power/date/work).
- **No backend, fully private**: data stays in the browser (localStorage + IndexedDB), no accounts, no server.
- **PWA that runs offline**, bilingual VI/EN.
- **Uses real map data**: OpenStreetMap (tiles + discovery), Nominatim (geocoding), OSRM (routing).
- **Has automated tests + CI/CD** — a big differentiator from many small projects.

## 1.2. What problem does it solve?

1. **Google Maps doesn't remember your "taste"** → Quán quen saves personal impressions across multiple criteria.
2. **Indecisive about "where to go today"** → suggestions by mood (work/date/quiet/quick bite/tasty/nearby/never-tried).
3. **Visiting several places in one outing** → plan an itinerary, draw the real route + time, open multi-stop directions.
4. **Want privacy** → all data on your device, sent nowhere (unless you enable Gist sync).
5. **Demonstrating technical ability**: interactive maps, advanced storage (IndexedDB), encryption, integrating multiple external services, **and engineering discipline** (automated tests + CI/CD) — the point-scorer.

> 💡 **Interview tip:** this project stands out for **technical maturity**: not just features, but 47 Playwright tests, encoding checks, and a CI/CD pipeline that deploys only when everything passes. Emphasize this — it shows you care about quality, not just "it works".

## 1.3. Main features

| Group | Features |
|-------|----------|
| **Map** | Leaflet + OSM tiles, markers scored by fit, clustering (markercluster), density heatmap |
| **Real discovery** | "Explore this area" / "Random nearby" scanning OpenStreetMap (Overpass), filter by type, radius 1-10km |
| **Routing** | Multi-stop itinerary, real routes via OSRM (straight-line fallback), open Google/Apple Maps directions |
| **Filter & search** | By type/taste/tag/area/price/distance/open-now; 5 sort modes; free search |
| **Place CRUD** | Add/edit/delete, rate 5 criteria, opening hours, visit history, favorite, **undo** |
| **Photos** | Multiple, compressed, albums (food/space/menu/other), stored in **IndexedDB**, lightbox, orphan cleanup |
| **Organization** | Custom collections, tag management (bulk rename/delete), custom place types |
| **Suggestions & stats** | "Where to go today?" by mood, stale-favorite reminders, visit + 6-month spending stats |
| **Import/export** | Google Maps link import, CSV in/out, plain + **encrypted (AES-GCM)** JSON backup, GitHub Gist sync |
| **UI** | Dark/light mode, bilingual VI/EN, responsive, toasts with action buttons |
| **PWA & offline** | manifest + service worker, cache app shell + tiles, share_target to receive links from phones |
| **Quality** | 47 Playwright tests, UTF-8 encoding check, GitHub Actions CI/CD |

## 1.4. Tech stack

**Core (client):**
- **HTML5 + CSS3 + plain JavaScript (ES2020)** — no framework, **no bundler/build step**.
- **Leaflet 1.9.4** — maps; **Leaflet.markercluster 1.5.3** — clustering; **leaflet.heat 0.2.0** — heatmap.
- **Lucide 1.17.0** — icons (`lucide.createIcons()`).
- Libraries loaded via **CDN unpkg, pinned to fixed versions** (never `@latest`).

**Storage:**
- **localStorage** — place data (text), config, collections, itineraries.
- **IndexedDB** — place images (base64), kept separate so localStorage doesn't fill up.
- **Web Crypto API** — encrypted backup (PBKDF2-SHA-256 250k iterations + AES-GCM).

**External web services (free):**
- **OpenStreetMap tiles** (map base), **Nominatim** (geocode/reverse), **Overpass API** (find real places), **OSRM** (routing), **GitHub Gist API** (optional sync).

**Dev / Test / CI:**
- **Node + Playwright** (47 E2E tests), **http-server** (local), **GitHub Actions** (CI: check + test → deploy GitHub Pages).
- `check-encoding.js` validates UTF-8 to avoid mojibake.

> 💡 **A trap question: "Why no framework for something this complex?"** Answer: "It's complex in *features* but the state architecture is simple enough for vanilla (one `render()` reads state and redraws the UI). No bundler means **deploy = push static files**, easy to host on GitHub Pages, and **Playwright tests call app functions directly** via `page.evaluate` since everything is in global scope. The cost is managing state/DOM myself — offset by 47 automated tests to keep quality."

## 1.5. How to run

```powershell
cd quan-quen-map
npm run serve        # http-server at http://127.0.0.1:5178
# or: python -m http.server 5178
```

Check & test:
```powershell
npm run check        # UTF-8 encoding + node --check app.js/i18n.js/sw.js + parse manifest
npm install          # first time
npx playwright install
npm test             # 47 Playwright tests (auto-starts the server)
```

> The service worker only runs over `http(s)://` (localhost or GitHub Pages), not via `file://`.

## 1.6. Folder structure

```
quan-quen-map/
├── index.html          # HTML skeleton + all static DOM; loads i18n.js → app.js
├── app.js              # All logic (~4,300 lines)
├── i18n.js             # VI/EN dictionary (I18N_STRINGS)
├── style.css           # CSS, theme vars, dark mode, responsive (~2,710 lines)
├── sw.js               # Service worker (cache app shell + tiles + runtime)
├── manifest.json       # PWA manifest (icons, share_target)
├── scripts/            # check-encoding.js, screenshots.js, gif.js (dev)
├── tests/app.spec.js   # 47 Playwright tests
├── playwright.config.js
├── package.json        # scripts: serve / test / check
└── .github/workflows/ci.yml   # CI: check + test → deploy Pages
```

Scale: ~8,800 lines (app.js ~4,718, style.css ~2,710, i18n.js ~688, index.html ~574, sw.js ~148).

> ⚠️ **External-services note:** Overpass/OSRM/Nominatim are shared free services that can be slow or rate-limited (handled by `fetchWithRetry` on 429/503 plus user notices). The places shown by default are **saved** places; real places appear only when you tap "Explore".
