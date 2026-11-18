# 05 — Web App Detail (Stage 3)

The web app packages the technical core into a tool for non-technical artists. It's a **FastAPI backend** plus a **vanilla HTML/CSS/JS frontend**.

---

## A. Backend — `backend/main.py`

### Setup and CORS
```python
app = FastAPI(title="Art Shield", version="0.1.0")
app.add_middleware(CORSMiddleware, allow_origins=["*"], ...)
```
- **CORS** open to `*` for easy dev/demo. (In an interview, say plainly: production should restrict origins.)

### Importing the `shield` package from the parent dir
```python
BASE_DIR = Path(__file__).resolve().parent.parent
if str(BASE_DIR) not in sys.path:
    sys.path.insert(0, str(BASE_DIR))
from shield.pipeline import protect_image, run_robustness
```
- The backend lives in `backend/`, while the `shield/` package is at the project root. We insert `BASE_DIR` into `sys.path` to import it. → A project-structure trick worth knowing.

### `_load_and_fit(upload)`
Read the upload with PIL, convert to RGB; if the longest side > `MAX_IMAGE_SIDE` (512), shrink keeping aspect ratio with `Image.LANCZOS`. Catches read errors → returns HTTP 400 with a clear message.

### `GET /api/health`
Returns `{status, device (cpu/cuda), presets}`. Useful to check the server is alive and whether it's on CPU or GPU.

### `POST /api/protect`
```python
async def protect(image: UploadFile, preset: str = Form("vua")):
    if preset not in PRESETS: raise HTTPException(400, ...)
    original = _load_and_fit(image)
    params = PRESETS[preset]
    protected, metrics = protect_image(original, **params, use_eot=True, use_perceptual=True)
    uid = uuid.uuid4().hex[:12]
    original.save(WEB_STORAGE_DIR / f"{uid}_original.png")
    protected.save(WEB_STORAGE_DIR / f"{uid}_protected.png")
    return JSONResponse({uid, preset, metrics, original_url, protected_url})
```
- Take image + preset name, apply the preset's params, call the core.
- Generate a random `uid`, save **both the (fitted) original and the protected image** — the original is needed for the verify step's cosine.
- Return image URLs for the frontend.

### `POST /api/verify`
Takes a `uid`, reads back the two saved images, calls `run_robustness()`, returns the result table. If the session isn't found → HTTP 404.

### Static serving (order matters)
```python
app.mount("/files", StaticFiles(directory=WEB_STORAGE_DIR))     # session images
app.mount("/", StaticFiles(directory=FRONTEND_DIR, html=True))  # the UI — LAST
```
Mount `/` last so StaticFiles doesn't hide the `/api/*` routes.

---

## B. Config — `backend/config.py`

```python
MAX_IMAGE_SIDE = 512
PRESETS = {
    "nhe":  {"epsilon": 0.04, "steps": 60,  "mask_min": 0.12, "eot_samples": 4},
    "vua":  {"epsilon": 0.06, "steps": 90,  "mask_min": 0.12, "eot_samples": 4},
    "manh": {"epsilon": 0.09, "steps": 140, "mask_min": 0.12, "eot_samples": 5},
}
```
- **Presets** turn technical params into 3 user-friendly levels (Light/Medium/Strong).
- `epsilon` rises (stronger noise = sturdier protection but more visible); `steps`/`eot_samples` rise too → stronger but slower.
- `WEB_STORAGE_DIR` is `mkdir(parents=True, exist_ok=True)` at import → auto-creates the storage dir.

---

## C. Frontend — `frontend/`

Three plain files, no framework:

### `index.html` — 4 cards
1. **Choose image**: drag-drop zone (`#dropzone`) + hidden file input.
2. **Protection level**: 3 radios Light/Medium/Strong (Medium `checked` by default).
3. **Results** (hidden initially): original/protected comparison, metrics, download + "Check robustness" buttons.
4. **Note**: reminder that protection isn't absolute.
There's an `#overlay` (spinner) shown while processing.

### `app.js` — logic
- **Drag-drop**: listens for `dragover`/`drop`/`change`, reads the file with `FileReader` to preview inside the dropzone.
- **Protect** (`protectBtn`):
  ```js
  const form = new FormData();
  form.append("image", selectedFile);
  form.append("preset", preset);
  const res = await fetch("/api/protect", { method: "POST", body: form });
  ```
  Shows the overlay while waiting (1–2 min), parses JSON, calls `renderResult`.
- **`renderResult`**: sets image `src` (with `?t=Date.now()` to **bust the cache**), colors the cosine by `m.protected` (green if met, red if not), shows SSIM/PSNR.
- **Check robustness** (`verifyBtn`): sends `uid`, gets the table, `renderVerify` builds a `<table>` with "🛡️ still protected" / "❌ broken" tags.
- **Overlay**: `showOverlay/hideOverlay` toggle the overlay and update its text.

### `style.css`
Dark theme, card layout, 2-column image comparison grid, "metric boxes", responsive (1 column on narrow screens), spinner animation. No CSS libraries.

---

## D. End-to-end UX

```
Drop image ─► pick "Medium" ─► click "Protect"
   └─► overlay "Adding protection..." (~1-2 min on CPU)
        └─► show before/after + cosine/SSIM/PSNR + Download
             └─► (optional) "Check robustness" ─► table of 9 wash-off transforms
```

## E. Web-side talking points

1. **Core separated from UI**: the web is a thin "front" calling `pipeline.py`; the CLI calls the same core. → easy to maintain and test.
2. **Presets instead of exposing epsilon/steps**: reduce cognitive load for artists.
3. **Image cache busting** via `?t=timestamp` — small detail that avoids the "image not updating" bug.
4. **Async + overlay**: since processing is slow, the UI reports status clearly instead of freezing silently.
5. **Honest security**: no auth yet, CORS open — fine for a demo, but knowing to flag it is a plus.
