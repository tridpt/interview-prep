# 7. Cheat Sheet — Quick Review Before Go-Time

Skim for 5-10 minutes before the interview.

---

## 🎯 30-second pitch (memorize)
> Quán quen: a **PWA** web app to save food/cafe spots **by personal taste** on a map — rate 5 personal criteria, suggest "where to go today" by mood, plan itineraries with real routes, discover new places from OpenStreetMap. **Vanilla JS + Leaflet, no backend, no bundler**. Data in **localStorage (text) + IndexedDB (images)**, **AES-GCM encrypted backups**, optional **GitHub Gist** sync. With **47 Playwright tests + CI/CD** (deploy only on pass), bilingual VI/EN, works offline.

## 🧱 Tech stack
- **Core**: Vanilla HTML/CSS/JS (ES2020), no bundler.
- **Maps**: Leaflet 1.9.4 + markercluster + leaflet.heat; Lucide icons (pinned CDN).
- **Storage**: localStorage (text) + IndexedDB (images) + Web Crypto (backup).
- **Services**: OSM tiles, Nominatim, Overpass, OSRM, GitHub Gist.
- **Test/CI**: Playwright (47 tests) + http-server + GitHub Actions.

## 🗂️ Files
| File | Role |
|---|---|
| index.html | static DOM; loads i18n.js → app.js + CDN |
| i18n.js | I18N_STRINGS vi/en |
| app.js (~4718) | brain: state, init, render, filter/score, CRUD, map, discovery, routing, backup |
| style.css (~2279) | theme + dark mode + responsive |
| sw.js | service worker, 4 cache strategies |
| tests/app.spec.js | 47 Playwright tests |

## 📐 Architecture
Client-only, **no backend**. Central **render()**: state → UI. localStorage (place text) + IndexedDB (images). Calls map services via fetch (with fetchWithRetry).

## 💾 Two-tier storage (KNOW IT)
Heavy images → **IndexedDB**; text → **localStorage** (images only {id,category}, NO dataUrl). Startup: loadImagesFromDb → imageCache → hydratePlaceImages. pruneOrphanImages cleans orphans. Reason: localStorage ~5MB.

## 🗺️ Leaflet
tileLayer (OSM, z/x/y) · marker + popup · markercluster (clustering) · leaflet.heat (heatmap). map.invalidateSize() on container resize.

## 🌐 External services
OSM tiles · Nominatim (address↔coords) · Overpass (find places: `node["amenity"~...]["name"](around:r,lat,lng)`) · OSRM (routing) · Gist (sync).
**fetchWithRetry**: 429/503 → wait then retry → rateLimited flag. Fallback: OSRM fails→straight line; discovery fails→saved list.

## 🔐 Encrypted backup
PBKDF2-SHA-256 **250,000 iterations** + salt → key; **AES-GCM** + random iv. File: salt/iv/data base64. Web Crypto API.

## 🧠 render() pipeline
getFilteredPlaces() → renderList/Markers/Heat/Route/Itinerary/Recommendations/Stats. Markers **reused** (diff the Map of ids, only add/remove/update the difference).

## 🎯 Mood suggestions
getMoodScore(place, mood): work/date/quiet/taste/nearby/new/quick. Reroll = **weighted random** (score^1.25, recently visited ×0.22).

## 🧪 Tests & CI (HIGHLIGHT)
**47 Playwright** E2E; call global functions via page.evaluate; network tests **mock window.fetch**. check-encoding.js against mojibake. **GitHub Actions**: check + test → deploy Pages only on pass.

## 📦 PWA
manifest + sw.js. Cache: app shell + CDN + tiles (cache-first, trim 240). 4 origin-based strategies. **share_target** receives links from phones. Install / Update buttons (SKIP_WAITING).

## 🔐 Security & privacy
Data on-device, sent nowhere (except Gist). escapeHtml all data (incl. OSM). Custom modal/toast, no native prompt(). Pinned CDN versions.

## ⚖️ Trade-offs
No-backend → private/simple, at the cost of manual sync (Gist). No-bundler → easy deploy, tests call global functions, at the cost of a large app.js.

## 🚀 Future improvements
Split into ES modules + bundler; a11y tests with a screen reader; cache Overpass by grid tiles; optional backend for real-time sync.

## 💬 Key terms
- **PWA / Service Worker / Cache API / share_target**
- **localStorage vs IndexedDB / imageCache / hydrate / orphan prune**
- **Leaflet / tile / marker / markercluster / heatmap**
- **Nominatim / Overpass / OSRM / Gist / POI / amenity**
- **fetchWithRetry / 429 / fallback / resilience**
- **Web Crypto / PBKDF2 / AES-GCM / salt / iv**
- **render() = f(state) / undo stack / normalizePlace**
- **Playwright E2E / mock fetch / GitHub Actions CI/CD**

---

### ✅ Pre-interview checklist
- [ ] 30-second pitch.
- [ ] Explain localStorage + IndexedDB & the image flow.
- [ ] Describe 47 tests + CI/CD blocking broken deploys.
- [ ] Encrypted backup (PBKDF2 + AES-GCM).
- [ ] fetchWithRetry + network fallback.
- [ ] render() = f(state).
- [ ] 2 limitations + fixes.
- [ ] 2 questions for the interviewer.

**Breathe. You understand this project. Tell your story with confidence. 🍀**
