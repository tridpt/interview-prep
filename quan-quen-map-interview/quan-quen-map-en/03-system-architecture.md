# 3. System Architecture

## 3.1. High-level diagram

```
┌───────────────────────────────── BROWSER ─────────────────────────────────────┐
│                                                                                │
│  index.html ──▶ static DOM + data-i18n                                         │
│  i18n.js   ──▶ I18N_STRINGS (vi/en) ──▶ t()                                    │
│  app.js    ──▶ init() ─▶ global state ─▶ render() ─▶ DOM + Leaflet              │
│  style.css ──▶ theme (light/dark) + responsive                                 │
│        │              │                  │                                     │
│        ▼              ▼                  ▼                                     │
│  localStorage    IndexedDB          Leaflet map                                │
│  (place text,    (place images)     (tiles, markers, cluster, heat)            │
│   config)                                                                      │
│        │                                  │  (fetch, with fetchWithRetry)      │
│        ▼                                  ▼                                     │
│   sw.js (PWA cache)            Nominatim · Overpass · OSRM · GitHub Gist        │
└────────────────────────────────────────────────────────────────────────────────┘
```

**Core trait:** fully client-side, **no backend**. All logic + state in the browser. It differs from a static game in that data is **dynamic** (many external map services), storage is **two-tier** (localStorage + IndexedDB), and it has **automated tests + CI/CD**.

## 3.2. Layers & file responsibilities

| File | Role |
|------|------|
| `index.html` | Skeleton & all static DOM; loads `i18n.js` → `app.js` + CDN libs |
| `i18n.js` | The `I18N_STRINGS` dictionary (vi/en) — loaded **before** app.js |
| `app.js` | The brain: state, init, render, filter/score, CRUD, map, discovery, routing, backup, i18n, theme |
| `style.css` | Theme (CSS vars + dark mode), grid layout, responsive |
| `sw.js` | Service worker: cache app shell + tiles + runtime with 4 origin-based strategies |
| `manifest.json` | PWA: name, icons, `share_target` |
| `tests/app.spec.js` | 47 Playwright tests |
| `.github/workflows/ci.yml` | CI: check + test → deploy Pages |

> 💡 **No bundler:** `index.html` loads 3 JS files directly in order `i18n.js → app.js` + CDN. Every function in `app.js` is in **global scope** → Playwright calls them directly via `page.evaluate`.

## 3.3. Startup flow (`init()`) — study this

`app.js` registers `DOMContentLoaded → init()`. Order:

```
1. cacheElements()        — gather every DOM element by id into `els`
2. applyStoredLang()      — read saved language, translate static UI
3. applyStoredTheme()     — apply dark/light
4. loadTypes()            — load place types (default + custom)
5. setSidebarCollapsed()  — restore sidebar state
6. hydrateStaticControls()— build dynamic controls (type/taste filters, select, ratings, days)
7. syncRangeLabels(); places=loadPlaces(); collections=...; savedItineraries=...
8. initMap()              — init Leaflet, tiles, cluster/heat layer
9. bindEvents()           — wire ALL events
10. render() + renderDataPanel() + refreshIcons()
11. setupInstallPrompt(); registerServiceWorker(); handleSharedLaunch()
12. initImages()          — ASYNC: load images from IndexedDB → hydrate → persist → prune orphans
13. maybeShowDiscoverHint()— first-launch hint
```

**Notable:** `loadPlaces()` **seeds 5 sample places** (in Ho Chi Minh City) if there's no data. `initImages()` runs **async** last because IndexedDB is async.

## 3.4. Render pipeline — the central `render()`

The whole UI is redrawn from state through **one `render()`** (a "state → render" model, like React but manual):

```
render():
  filtered = getFilteredPlaces()        // apply all filters + sorting
  renderTagFilters / renderTagManager
  renderAreaFilters
  renderRecommendations                 // "where to go today" by mood
  renderFavoriteReminder                // remind stale favorites
  renderList(filtered)                  // the place list on the left
  renderMarkers(filtered)               // map markers
  renderHeat(filtered)                  // heatmap (if on)
  renderRoute / renderItinerary         // route + itinerary
  renderCollections / renderPlaceDetail / renderStats
  refreshIcons()                        // Lucide redraws icons
```

➡️ **Philosophy:** any state change (filter, select, add place...) → call `render()` → UI consistent with state. Simple, easy to reason about, no framework needed.

## 3.5. Data model

### Place
```js
{
  id, name, type,              // type: cafe|food|sweet|drink|c-<custom>
  address, lat, lng,
  priceLevel: 1..4,
  ratings: { taste, quiet, power, date, work },  // each 0..5
  tags: string[],              // max 12
  notes, lastVisit, favorite,
  images: [{ id, category }],  // category: food|space|menu|other — NO dataUrl in localStorage
  openingHours: { days:[0..6], open:"HH:MM", close:"HH:MM" },
  visits: [{ id, date, rating, note }],
  createdAt, updatedAt
}
```
- `normalizePlace()` normalizes every place (defaults, valid filtering); `isValidPlace()` requires a name + finite lat/lng.
- Images: in RAM have `dataUrl`; in localStorage only `{id, category}`; in IndexedDB `id → dataUrl`.

### Storage keys (localStorage)
| Key | Content |
|-----|---------|
| `quan-quen-map:places:v1` | Places (images stripped) |
| `:collections:v1` `:itineraries:v1` `:types:v1` | Collections / itineraries / custom types |
| `:theme:v1` `:lang:v1` `:sidebar-state:v1` | UI preferences |
| `:gist-settings:v1` `:backup-meta:v1` | Sync & backup |
| `:discover-hint:v1` | Whether the first-launch hint was shown |

**IndexedDB:** DB `quan-quen-map-images`, store `images` (key = image id, value = dataUrl).

## 3.6. Global state (in app.js)

Main variables: `places`, `selectedId`, `activeTypes/activePurposes/activeTags/activeAreas` (all `Set`s), `openNowOnly`, `priceRange`, `maxDistanceKm`, `itinerary`, `collections`, `savedItineraries`, `userLocation`, `map`, `markers` (Map), `markerLayer`, `heatLayer`, `discoverMarkers`, `imageCache` (Map), `lang`, `undoStack`, ...

## 3.7. The offline layer — Service Worker (4 strategies)

```
install  → precache APP_SHELL (local files + CDN Leaflet/lucide) → skipWaiting
activate → delete caches != CACHE_VERSION → clients.claim
message  → receive SKIP_WAITING (when user clicks "Update")
fetch    → classify by origin:
   Nominatim/Overpass/OSRM  → fetch directly (no cache)
   tile.openstreetmap.org   → cache-first (capped at 240 tiles, trimCache)
   navigate (page)          → network-first
   app shell                → stale-while-revalidate
   else (runtime)           → stale-while-revalidate
```
Changing assets requires **bumping `CACHE_VERSION`** (e.g. v36 → v37). `Promise.allSettled` during precache so one failed file doesn't break install.

## 3.8. Why is this architecture sound?

- **Private & offline-first**: data on-device, works offline with saved places.
- **Two-tier storage**: text in localStorage (fast, simple), images in IndexedDB (large) → no 5MB limit hit.
- **Defensive dependence on shared services**: `fetchWithRetry` + fallbacks (OSRM fails → straight line; discovery fails → suggest from saved list).
- **Quality assured**: 47 tests + CI blocking broken deploys — rare in a personal project, a major interview highlight.
