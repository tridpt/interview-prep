# 03 — System Architecture

## 1. Big picture

```
┌──────────────────────────────┐         ┌─────────────────────────────┐
│  FRONTEND (HTML/CSS/JS)       │         │  BACKEND (FastAPI)          │
│                              │  HTTP   │                             │
│  Tab 1: Recording studio      │ ◄─────► │  main.py                    │
│   - wav-recorder.js (mic)     │  /api   │   /api/sentences            │
│   - upload per sentence       │         │   /api/record (POST/DELETE) │
│  Tab 2: Try cloned voice       │         │   /api/dataset/download     │
│   - record/upload sample       │         │   /api/tts ──┐              │
│   - type text → listen         │         │              ▼              │
└──────────────────────────────┘         │   tts_engine.py (viXTTS)    │
                                          │   (lazy-load, pluggable)    │
                                          └──────────┬──────────────────┘
                                                     ▼
                                   storage/ (dataset wavs + outputs)
                                   Hugging Face (download viXTTS weights)
```

## 2. Layers and responsibilities

| Layer | File | Responsibility |
|-------|------|----------------|
| **Web I/O** | `backend/main.py` | API routes, save files, zip packaging, validate consent |
| **TTS engine** | `backend/tts_engine.py` | Load viXTTS (lazy), clone voice, synthesize WAV |
| **Config** | `backend/config.py` | Paths, device (cpu/cuda), HF repo, limits |
| **Data** | `backend/sentences.py` | 200 Vietnamese sample sentences |
| **UI** | `frontend/index.html`, `app.js`, `style.css` | 2 tabs, display, call API |
| **Recording** | `frontend/wav-recorder.js` | Mic → 16-bit PCM mono WAV |
| **Fine-tune** | `finetune/` | Colab notebook to train a custom voice |

> Principle: the backend is **thin** (just orchestration + file saving); the heavy part (model) is wrapped in `tts_engine.py` as pluggable; the complex recording is pushed to the **client** so the backend needs no heavy audio libraries.

## 3. Two tabs — two flows

### Tab 1: Dataset studio
```
GET /api/sentences → list of 200 sentences + which are recorded
[each sentence] click "Record" → wav-recorder records mic → "Stop & save"
  → POST /api/record {sentence_id, audio(wav)} → save 0001.wav, rebuild metadata.csv
[delete] DELETE /api/record/{id} → delete wav, rebuild metadata
[download] GET /api/dataset/download → zip {metadata.csv + wavs/} → download
```

### Tab 2: Try cloned voice
```
(optional) upload a voice sample, or use a sample recorded in Tab 1
type text + check consent → click "Generate"
  → POST /api/tts {text, consent, reference_id, audio?}
    → backend checks consent, text length
    → _resolve_reference: prefer upload > reference_id > first sample
    → engine.synthesize(text, reference_wav) → WAV
  → return FileResponse(wav) → play + allow download
```

## 4. Endpoints

| Method | Path | Role |
|--------|------|------|
| GET | `/api/health` | Status + device + recorded count |
| GET | `/api/sentences` | 200 sentences + `recorded` flag |
| POST | `/api/record` | Save one clip (by sentence_id) |
| DELETE | `/api/record/{id}` | Delete one clip to re-record |
| GET | `/api/dataset/download` | Package dataset.zip |
| POST | `/api/tts` | Generate cloned voice (consent required) |
| (mount) | `/storage`, `/` | Audio files + static frontend |

## 5. Pluggable engine — the most important design

```python
class ViXTTSEngine:
    def synthesize(self, text, reference_wav, language="vi") -> str:
        ...  # return a .wav path
engine = ViXTTSEngine()   # shared instance
```
- `main.py` just calls `engine.synthesize(...)`, **not knowing** whether it's viXTTS or another model.
- To swap models: write a new class with the same method and reassign `engine`. → decouples the backend from the specific model.

## 6. Lazy-load + auto-download weights

```python
def _load(self):
    if self._model is not None: return     # already loaded
    model_dir = self._ensure_downloaded()  # download from HF if missing
    config = XttsConfig(); config.load_json(...)
    model = Xtts.init_from_config(config)
    model.load_checkpoint(config, checkpoint_dir=model_dir, ...)
    model.to(DEVICE); model.eval()
    self._model = model
```
- The model loads only on the first `synthesize` → fast startup.
- `_ensure_downloaded`: checks `config.json/model.pth/vocab.json`; if missing, `snapshot_download` from Hugging Face.
- Imports `TTS.tts...` **inside** `_load` so the app still boots without the TTS library (clear error only when actually needed).

## 7. Auto device detection (CPU/GPU)

```python
def _detect_device():
    try:
        import torch
        return "cuda" if torch.cuda.is_available() else "cpu"
    except ImportError:
        return "cpu"
DEVICE = _detect_device()
```
- Uses GPU if available (Colab/CUDA machine), else CPU.
- Soft torch import: the app still boots without torch installed.

## 8. Why record + encode WAV on the client?

- The backend `/api/record` only needs to **save the file** (`shutil.copyfileobj`), no heavy audio library → a lean backend.
- The complex part (mic recording, downsample, WAV header) is done in JS in the browser (`wav-recorder.js`).
- Result: 16-bit PCM mono WAV in exactly the format the fine-tune pipeline needs.

## 9. Storage

```
storage/
├── dataset/
│   ├── metadata.csv      # regenerated after each record/delete
│   └── wavs/0000.wav...  # recorded clips (named by sentence_id)
├── uploads/              # uploaded reference voice (ref_tmp.wav)
└── outputs/              # generated result wavs (uuid.wav)
```

## 10. Core design principles

1. **Thin backend, thick engine**: orchestration is separate from the model; the model is pluggable.
2. **Lazy + soft imports**: fast startup, resilient to missing heavy libraries.
3. **Push audio processing to the client**: the backend needs no ffmpeg/librosa to accept recordings.
4. **Consent is the gate**: `/api/tts` refuses if `consent=False`.
5. **Runs on CPU, faster on GPU**: auto device detection, recommend Colab for fine-tuning.
