# 04 — Backend Deep Dive (function by function)

The backend has four files: `config.py`, `sentences.py`, `main.py`, `tts_engine.py`.

---

## A. `config.py` — centralized config

```python
BASE_DIR = Path(__file__).resolve().parent.parent
MODEL_DIR = BASE_DIR / "backend" / "model"   # viXTTS weights
STORAGE_DIR = BASE_DIR / "storage"
UPLOAD_DIR = STORAGE_DIR / "uploads"
OUTPUT_DIR = STORAGE_DIR / "outputs"
for _d in (MODEL_DIR, UPLOAD_DIR, OUTPUT_DIR):
    _d.mkdir(parents=True, exist_ok=True)     # auto-create dirs

HF_REPO_ID = "capleaf/viXTTS"
DEFAULT_LANGUAGE = "vi"
DEVICE = _detect_device()                      # cuda if available, else cpu
MAX_TEXT_LENGTH = 1000                          # max chars per synthesis
MIN_REFERENCE_SECONDS = 3                       # recommended sample length
```
- All paths/params in one place → easy to tweak.
- `_detect_device()` soft-imports torch so the app boots even without torch.
- Auto-`mkdir`s the storage dirs at import.

## B. `sentences.py` — 200 sample sentences

`SAMPLE_SENTENCES` is a list of 200 Vietnamese sentences, **carefully chosen**:
- Cover the **6 tones**, many initial/final consonants, vowels.
- Include sentences with **numbers, dates, measurements**, questions, exclamations → natural voice.
- Include **hard-to-pronounce** words (khúc khuỷu, ngoằn ngoèo...) so the model learns robustly.
- 2–15 seconds when read → fit to cut into training clips.
> A "behind-the-scenes" but important part: fine-tuned voice quality depends heavily on whether the dataset covers enough phonemes.

## C. `main.py` — API

### Setup
```python
app = FastAPI(title="Vietnamese Voice Clone Studio", version="0.2.0")
app.add_middleware(CORSMiddleware, allow_origins=["*"], ...)
DATASET_DIR = STORAGE_DIR / "dataset"; WAVS_DIR = DATASET_DIR / "wavs"
WAVS_DIR.mkdir(parents=True, exist_ok=True)
METADATA_PATH = DATASET_DIR / "metadata.csv"
```

### `GET /api/health`
Returns `{status, device, total_sentences, recorded}` — recorded = number of wav files. Useful to check device and progress.

### `GET /api/sentences`
```python
recorded_ids = {p.stem for p in WAVS_DIR.glob("*.wav")}
items = [{"id": i, "text": text, "recorded": f"{i:04d}" in recorded_ids}
         for i, text in enumerate(SAMPLE_SENTENCES)]
```
Returns the 200 sentences with a `recorded` flag (based on the `0001.wav` filename).

### `POST /api/record`
```python
async def record_sample(sentence_id: int = Form(...), audio: UploadFile = File(...)):
    if not (0 <= sentence_id < len(SAMPLE_SENTENCES)): raise HTTPException(400, ...)
    wav_path = WAVS_DIR / f"{sentence_id:04d}.wav"
    with wav_path.open("wb") as f: shutil.copyfileobj(audio.file, f)  # just SAVE
    _rebuild_metadata()
    return {"ok": True, "recorded": ..., "total": ...}
```
- Audio is already WAV (encoded on the client) so the backend **just saves the file** — no audio library.
- Named by `sentence_id` (4 digits) → overwrites if re-recording that sentence.

### `DELETE /api/record/{sentence_id}`
Deletes the corresponding wav (if any), then rebuilds metadata → allows re-recording.

### `_rebuild_metadata()`
```python
for p in sorted(WAVS_DIR.glob("*.wav")):
    idx = int(p.stem)
    rows.append([f"wavs/{p.name}", SAMPLE_SENTENCES[idx], "speaker1"])
csv.writer(f, delimiter="|").writerows(rows)
```
Regenerates `metadata.csv` (LJSpeech-style) from current wavs → always in sync with actual files.

### `GET /api/dataset/download`
```python
buf = io.BytesIO()
with zipfile.ZipFile(buf, "w", ZIP_DEFLATED) as zf:
    zf.write(METADATA_PATH, "dataset/metadata.csv")
    for p in wavs: zf.write(p, f"dataset/wavs/{p.name}")
return StreamingResponse(buf, media_type="application/zip",
    headers={"Content-Disposition": "attachment; filename=dataset.zip"})
```
- Packages the dataset **in memory** (BytesIO) and streams it — no temp zip on disk.
- Returns 400 if nothing has been recorded.

### `POST /api/tts` — generate cloned voice
```python
async def try_voice(text=Form(...), consent=Form(...), reference_id=Form(-1), audio=File(None)):
    if not consent: raise HTTPException(400, "You must confirm rights to use...")
    text = text.strip()
    if not text: raise HTTPException(400, ...)
    if len(text) > MAX_TEXT_LENGTH: raise HTTPException(400, "too long...")
    ref_path = _resolve_reference(audio, reference_id)
    try:
        from tts_engine import engine                 # lazy import
        out_path = engine.synthesize(text=text, reference_wav=str(ref_path))
    except RuntimeError as exc: raise HTTPException(503, str(exc))   # TTS not installed
    except Exception as exc: raise HTTPException(500, f"Synthesis error: {exc}")
    return FileResponse(out_path, media_type="audio/wav", filename="cloned_voice.wav")
```
- **Consent required** first.
- Validate empty / too-long text (`MAX_TEXT_LENGTH=1000`).
- `import engine` here (lazy) → app isn't heavy at startup.
- Distinguish errors: `RuntimeError` (TTS not installed) → 503; others → 500. Clear messages.

### `_resolve_reference(audio, reference_id)` — pick the sample
Priority order:
1. **Uploaded file** (if any) → save `uploads/ref_tmp.wav`.
2. **reference_id ≥ 0** → use `wavs/{id}.wav` if it exists.
3. **Fallback**: the **first** recorded sample in the dataset.
4. If none → HTTP 400 instructing to record/upload a voice.

### Static mounts
```python
app.mount("/storage", StaticFiles(directory=STORAGE_DIR))      # replay audio
app.mount("/", StaticFiles(directory=frontend_dir, html=True)) # the UI — LAST
```

---

## D. `tts_engine.py` — viXTTS wrapper (the AI core)

### Class structure
```python
class ViXTTSEngine:
    def __init__(self):
        self._model = None       # lazy
        self._normalizer = None
engine = ViXTTSEngine()          # shared instance for the whole app
```

### `_ensure_downloaded()`
Checks for `config.json, model.pth, vocab.json` in `MODEL_DIR`; if missing, `snapshot_download(repo_id="capleaf/viXTTS", local_dir=MODEL_DIR)` from Hugging Face. Returns the model dir.

### `_load()` — load the model (once)
```python
if self._model is not None: return
from TTS.tts.configs.xtts_config import XttsConfig   # import inside the function
from TTS.tts.models.xtts import Xtts
model_dir = self._ensure_downloaded()
config = XttsConfig(); config.load_json(model_dir/"config.json")
model = Xtts.init_from_config(config)
model.load_checkpoint(config, checkpoint_dir=model_dir, use_deepspeed=False)
model.to(DEVICE); model.eval()
self._model = model
```
- Imports `TTS...` **inside the function** → if the library is missing, raise `RuntimeError` with install instructions instead of crashing at startup.
- `model.eval()` + load onto `DEVICE` (cpu/cuda).

### `_normalize_vi(text)` — Vietnamese text normalization
```python
try:
    from vinorm import TTSnorm
    self._normalizer = TTSnorm
except Exception:
    self._normalizer = lambda t, **_: t   # FALLBACK: return text as-is
return self._normalizer(text, unknown=False, lower=False, rule=True)
```
- Normalizes numbers/symbols into words for natural reading (e.g. "15%" → "fifteen percent").
- `vinorm` often fails to build on Python 3.13 → **soft fallback**: if missing, use the raw text, engine still runs.

### `synthesize(text, reference_wav, language)` — the heart
```python
self._load()
text = self._normalize_vi(text.strip())
gpt_cond_latent, speaker_embedding = self._model.get_conditioning_latents(
    audio_path=reference_wav, gpt_cond_len=..., max_ref_length=..., sound_norm_refs=...)
out = self._model.inference(
    text=text, language=language,
    gpt_cond_latent=gpt_cond_latent, speaker_embedding=speaker_embedding,
    temperature=0.3, enable_text_splitting=True)
out_path = OUTPUT_DIR / f"{uuid.uuid4().hex}.wav"
self._save_wav(out["wav"], out_path)
return str(out_path)
```
Points to explain:
- **`get_conditioning_latents`**: extracts `(gpt_cond_latent, speaker_embedding)` from the sample — the "learn the voice" step.
- **`inference`**: generates the waveform reading `text` in that voice.
- **`temperature=0.3`**: low → stable voice, less improvisation (good for clear reading).
- **`enable_text_splitting=True`**: auto-splits long text into chunks for steadier reading.
- Saves with a **uuid** name → avoids collisions between generations.

### `_save_wav(wav, path)`
```python
tensor = torch.tensor(wav).unsqueeze(0)
torchaudio.save(str(path), tensor, sample_rate=24000)
```
viXTTS outputs at **24000 Hz** → saved as WAV via torchaudio.

---

## Backend spirit, summarized
> The backend is **thin and fault-tolerant**: it only saves files / zips / orchestrates, validating consent + length. The AI lives in `tts_engine.py` following the **lazy + pluggable + soft-import** pattern, extracting conditioning latents to clone the voice and synthesize 24kHz WAV. Every heavy import (TTS, torch, vinorm) is "soft" so the app boots in any environment.
