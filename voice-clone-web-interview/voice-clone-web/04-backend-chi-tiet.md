# 04 — Chi tiết Backend (đọc từng hàm)

Backend gồm 4 file: `config.py`, `sentences.py`, `main.py`, `tts_engine.py`.

---

## A. `config.py` — cấu hình tập trung

```python
BASE_DIR = Path(__file__).resolve().parent.parent
MODEL_DIR = BASE_DIR / "backend" / "model"   # weights viXTTS
STORAGE_DIR = BASE_DIR / "storage"
UPLOAD_DIR = STORAGE_DIR / "uploads"
OUTPUT_DIR = STORAGE_DIR / "outputs"
for _d in (MODEL_DIR, UPLOAD_DIR, OUTPUT_DIR):
    _d.mkdir(parents=True, exist_ok=True)     # tự tạo thư mục

HF_REPO_ID = "capleaf/viXTTS"
DEFAULT_LANGUAGE = "vi"
DEVICE = _detect_device()                      # cuda nếu có, không thì cpu
MAX_TEXT_LENGTH = 1000                          # ký tự tối đa mỗi lần sinh
MIN_REFERENCE_SECONDS = 3                       # độ dài audio mẫu khuyến nghị
```
- Mọi đường dẫn/tham số ở một chỗ → dễ chỉnh.
- `_detect_device()` import torch "mềm" để app khởi động được dù chưa cài torch.
- Tự `mkdir` các thư mục lưu trữ ngay khi import.

## B. `sentences.py` — 200 câu mẫu

`SAMPLE_SENTENCES` là list 200 câu tiếng Việt, **chọn lọc kỹ**:
- Phủ đủ **6 thanh điệu**, nhiều phụ âm đầu/cuối, nguyên âm.
- Có câu chứa **số, ngày tháng, đo lường**, câu hỏi, cảm thán → giọng tự nhiên.
- Có câu chứa **từ khó phát âm** (khúc khuỷu, ngoằn ngoèo...) để model học chắc.
- Độ dài 2–15 giây khi đọc → hợp cắt thành mẫu train.
> Đây là phần "ngầm" nhưng quan trọng: chất lượng giọng fine-tune phụ thuộc nhiều vào việc dataset có phủ đủ âm vị hay không.

## C. `main.py` — API

### Khởi tạo
```python
app = FastAPI(title="Vietnamese Voice Clone Studio", version="0.2.0")
app.add_middleware(CORSMiddleware, allow_origins=["*"], ...)
DATASET_DIR = STORAGE_DIR / "dataset"; WAVS_DIR = DATASET_DIR / "wavs"
WAVS_DIR.mkdir(parents=True, exist_ok=True)
METADATA_PATH = DATASET_DIR / "metadata.csv"
```

### `GET /api/health`
Trả `{status, device, total_sentences, recorded}` — recorded = số file wav đã thu. Hữu ích kiểm tra device và tiến độ.

### `GET /api/sentences`
```python
recorded_ids = {p.stem for p in WAVS_DIR.glob("*.wav")}
items = [{"id": i, "text": text, "recorded": f"{i:04d}" in recorded_ids}
         for i, text in enumerate(SAMPLE_SENTENCES)]
```
Trả 200 câu kèm cờ `recorded` — câu nào đã thu (dựa vào tên file `0001.wav`).

### `POST /api/record`
```python
async def record_sample(sentence_id: int = Form(...), audio: UploadFile = File(...)):
    if not (0 <= sentence_id < len(SAMPLE_SENTENCES)): raise HTTPException(400, ...)
    wav_path = WAVS_DIR / f"{sentence_id:04d}.wav"
    with wav_path.open("wb") as f: shutil.copyfileobj(audio.file, f)  # chỉ LƯU
    _rebuild_metadata()
    return {"ok": True, "recorded": ..., "total": ...}
```
- Audio đã là WAV (encode ở client) nên backend **chỉ lưu file** — không cần thư viện audio.
- Đặt tên theo `sentence_id` (4 chữ số) → ghi đè nếu thu lại câu đó.

### `DELETE /api/record/{sentence_id}`
Xóa file wav tương ứng (nếu có) rồi rebuild metadata → cho phép thu lại.

### `_rebuild_metadata()`
```python
for p in sorted(WAVS_DIR.glob("*.wav")):
    idx = int(p.stem)
    rows.append([f"wavs/{p.name}", SAMPLE_SENTENCES[idx], "speaker1"])
csv.writer(f, delimiter="|").writerows(rows)
```
Sinh lại `metadata.csv` (LJSpeech-style) từ các wav hiện có → luôn đồng bộ với file thực tế.

### `GET /api/dataset/download`
```python
buf = io.BytesIO()
with zipfile.ZipFile(buf, "w", ZIP_DEFLATED) as zf:
    zf.write(METADATA_PATH, "dataset/metadata.csv")
    for p in wavs: zf.write(p, f"dataset/wavs/{p.name}")
return StreamingResponse(buf, media_type="application/zip",
    headers={"Content-Disposition": "attachment; filename=dataset.zip"})
```
- Đóng gói dataset **trong bộ nhớ** (BytesIO) rồi stream về — không tạo file zip tạm trên đĩa.
- Báo lỗi 400 nếu chưa thu mẫu nào.

### `POST /api/tts` — sinh giọng clone
```python
async def try_voice(text=Form(...), consent=Form(...), reference_id=Form(-1), audio=File(None)):
    if not consent: raise HTTPException(400, "Bạn cần xác nhận quyền sử dụng...")
    text = text.strip()
    if not text: raise HTTPException(400, ...)
    if len(text) > MAX_TEXT_LENGTH: raise HTTPException(400, "quá dài...")
    ref_path = _resolve_reference(audio, reference_id)
    try:
        from tts_engine import engine                 # lazy import
        out_path = engine.synthesize(text=text, reference_wav=str(ref_path))
    except RuntimeError as exc: raise HTTPException(503, str(exc))   # chưa cài TTS
    except Exception as exc: raise HTTPException(500, f"Lỗi khi sinh: {exc}")
    return FileResponse(out_path, media_type="audio/wav", filename="cloned_voice.wav")
```
- **Consent bắt buộc** trước tiên.
- Validate text rỗng / quá dài (`MAX_TEXT_LENGTH=1000`).
- `import engine` ở đây (lazy) → app không nặng lúc khởi động.
- Phân biệt lỗi: `RuntimeError` (chưa cài TTS) → 503; lỗi khác → 500. Thông báo tiếng Việt rõ ràng.

### `_resolve_reference(audio, reference_id)` — chọn giọng mẫu
Thứ tự ưu tiên:
1. **File upload** (nếu có) → lưu `uploads/ref_tmp.wav`.
2. **reference_id ≥ 0** → dùng `wavs/{id}.wav` nếu tồn tại.
3. **Fallback**: mẫu thu âm **đầu tiên** trong dataset.
4. Nếu không có gì → HTTP 400 hướng dẫn thu/tải giọng.

### Mount tĩnh
```python
app.mount("/storage", StaticFiles(directory=STORAGE_DIR))      # nghe lại audio
app.mount("/", StaticFiles(directory=frontend_dir, html=True)) # giao diện — CUỐI
```

---

## D. `tts_engine.py` — Wrapper viXTTS (phần lõi AI)

### Cấu trúc class
```python
class ViXTTSEngine:
    def __init__(self):
        self._model = None       # lazy
        self._normalizer = None
engine = ViXTTSEngine()          # instance dùng chung toàn app
```

### `_ensure_downloaded()`
Kiểm tra `config.json, model.pth, vocab.json` trong `MODEL_DIR`; thiếu thì `snapshot_download(repo_id="capleaf/viXTTS", local_dir=MODEL_DIR)` từ Hugging Face. Trả về thư mục model.

### `_load()` — nạp model (1 lần)
```python
if self._model is not None: return
from TTS.tts.configs.xtts_config import XttsConfig   # import trong hàm
from TTS.tts.models.xtts import Xtts
model_dir = self._ensure_downloaded()
config = XttsConfig(); config.load_json(model_dir/"config.json")
model = Xtts.init_from_config(config)
model.load_checkpoint(config, checkpoint_dir=model_dir, use_deepspeed=False)
model.to(DEVICE); model.eval()
self._model = model
```
- Import `TTS...` **trong hàm** → nếu thiếu thư viện, raise `RuntimeError` với hướng dẫn cài, không làm sập app lúc khởi động.
- `model.eval()` + nạp lên `DEVICE` (cpu/cuda).

### `_normalize_vi(text)` — chuẩn hóa text tiếng Việt
```python
try:
    from vinorm import TTSnorm
    self._normalizer = TTSnorm
except Exception:
    self._normalizer = lambda t, **_: t   # FALLBACK: trả nguyên text
return self._normalizer(text, unknown=False, lower=False, rule=True)
```
- Chuẩn hóa số/ký hiệu sang chữ để đọc tự nhiên (vd "15%" → "mười lăm phần trăm").
- `vinorm` hay lỗi build trên Python 3.13 → **fallback mềm**: thiếu thì dùng text gốc, engine vẫn chạy.

### `synthesize(text, reference_wav, language)` — trái tim
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
Các điểm cần giải thích:
- **`get_conditioning_latents`**: trích `(gpt_cond_latent, speaker_embedding)` từ audio mẫu — bước "học giọng".
- **`inference`**: sinh sóng đọc `text` theo giọng đó.
- **`temperature=0.3`**: thấp → giọng ổn định, ít ngẫu hứng (hợp đọc rõ ràng).
- **`enable_text_splitting=True`**: tự tách câu dài thành đoạn để đọc ổn định hơn.
- Lưu kết quả với tên **uuid** → tránh trùng file giữa các lần sinh.

### `_save_wav(wav, path)`
```python
tensor = torch.tensor(wav).unsqueeze(0)
torchaudio.save(str(path), tensor, sample_rate=24000)
```
viXTTS xuất sóng ở **24000 Hz** → lưu WAV bằng torchaudio.

---

## Tóm tắt tinh thần backend
> Backend **mỏng và chịu lỗi**: chỉ lưu file/đóng zip/điều phối, validate consent + độ dài. Phần AI gói trong `tts_engine.py` theo mẫu **lazy + pluggable + soft import**, trích conditioning latents để clone giọng và sinh WAV 24kHz. Mọi import nặng (TTS, torch, vinorm) đều "mềm" để app khởi động được trong mọi môi trường.
