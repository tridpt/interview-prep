# 03 — Kiến trúc hệ thống

## 1. Bức tranh tổng thể

```
┌──────────────────────────────┐         ┌─────────────────────────────┐
│  FRONTEND (HTML/CSS/JS)       │         │  BACKEND (FastAPI)          │
│                              │  HTTP   │                             │
│  Tab 1: Studio thu âm         │ ◄─────► │  main.py                    │
│   - wav-recorder.js (micro)   │  /api   │   /api/sentences            │
│   - upload từng câu           │         │   /api/record (POST/DELETE) │
│  Tab 2: Thử giọng clone        │         │   /api/dataset/download     │
│   - thu/tải giọng mẫu          │         │   /api/tts ──┐              │
│   - nhập text → nghe           │         │              ▼              │
└──────────────────────────────┘         │   tts_engine.py (viXTTS)    │
                                          │   (lazy-load, pluggable)    │
                                          └──────────┬──────────────────┘
                                                     ▼
                                   storage/ (dataset wavs + outputs)
                                   Hugging Face (tải weights viXTTS)
```

## 2. Các lớp và trách nhiệm

| Lớp | File | Trách nhiệm |
|-----|------|-------------|
| **Giao tiếp web** | `backend/main.py` | Route API, lưu file, đóng gói zip, validate consent |
| **Engine TTS** | `backend/tts_engine.py` | Nạp viXTTS (lazy), clone giọng, sinh WAV |
| **Cấu hình** | `backend/config.py` | Đường dẫn, device (cpu/cuda), repo HF, giới hạn |
| **Dữ liệu** | `backend/sentences.py` | 200 câu mẫu tiếng Việt |
| **Giao diện** | `frontend/index.html`, `app.js`, `style.css` | 2 tab, hiển thị, gọi API |
| **Thu âm** | `frontend/wav-recorder.js` | Ghi micro → WAV 16-bit PCM mono |
| **Fine-tune** | `finetune/` | Notebook Colab huấn luyện giọng riêng |

> Nguyên tắc: backend **mỏng** (chỉ điều phối + lưu file), phần nặng (model) gói trong `tts_engine.py` dạng pluggable; phần thu âm phức tạp đẩy sang **client** để backend không cần thư viện xử lý audio nặng.

## 3. Hai tab — hai luồng

### Tab 1: Studio thu dataset
```
GET /api/sentences → danh sách 200 câu + câu nào đã thu
[mỗi câu] bấm "Thu" → wav-recorder ghi micro → "Dừng & lưu"
  → POST /api/record {sentence_id, audio(wav)} → lưu 0001.wav, rebuild metadata.csv
[xóa] DELETE /api/record/{id} → xóa wav, rebuild metadata
[tải] GET /api/dataset/download → zip {metadata.csv + wavs/} → tải về
```

### Tab 2: Thử giọng clone
```
(tùy chọn) tải file giọng mẫu, hoặc dùng mẫu đã thu ở Tab 1
nhập text + tích consent → bấm "Tạo giọng nói"
  → POST /api/tts {text, consent, reference_id, audio?}
    → backend kiểm consent, độ dài text
    → _resolve_reference: ưu tiên file upload > reference_id > mẫu đầu tiên
    → engine.synthesize(text, reference_wav) → WAV
  → trả FileResponse(wav) → phát + cho tải
```

## 4. Các endpoint

| Method | Đường dẫn | Vai trò |
|--------|-----------|---------|
| GET | `/api/health` | Trạng thái + device + số mẫu đã thu |
| GET | `/api/sentences` | 200 câu + cờ `recorded` |
| POST | `/api/record` | Lưu 1 đoạn thu (theo sentence_id) |
| DELETE | `/api/record/{id}` | Xóa 1 đoạn để thu lại |
| GET | `/api/dataset/download` | Đóng gói dataset.zip |
| POST | `/api/tts` | Sinh giọng clone (có consent) |
| (mount) | `/storage`, `/` | File audio + frontend tĩnh |

## 5. Engine pluggable — thiết kế quan trọng nhất

```python
class ViXTTSEngine:
    def synthesize(self, text, reference_wav, language="vi") -> str:
        ...  # trả về đường dẫn .wav
engine = ViXTTSEngine()   # instance dùng chung
```
- `main.py` chỉ cần gọi `engine.synthesize(...)`, **không biết** bên trong là viXTTS hay model khác.
- Muốn đổi model: viết class mới cùng phương thức rồi gán lại `engine`. → tách rời (decoupling) backend khỏi model cụ thể.

## 6. Lazy-load + tải weights tự động

```python
def _load(self):
    if self._model is not None: return     # đã nạp thì thôi
    model_dir = self._ensure_downloaded()  # tải từ HF nếu thiếu
    config = XttsConfig(); config.load_json(...)
    model = Xtts.init_from_config(config)
    model.load_checkpoint(config, checkpoint_dir=model_dir, ...)
    model.to(DEVICE); model.eval()
    self._model = model
```
- Model chỉ nạp khi `synthesize` được gọi lần đầu → app khởi động nhanh.
- `_ensure_downloaded`: kiểm tra `config.json/model.pth/vocab.json`; thiếu thì `snapshot_download` từ Hugging Face.
- Import `TTS.tts...` **bên trong** `_load` để app vẫn lên dù chưa cài thư viện TTS (báo lỗi rõ khi thực sự cần).

## 7. Tự phát hiện device (CPU/GPU)

```python
def _detect_device():
    try:
        import torch
        return "cuda" if torch.cuda.is_available() else "cpu"
    except ImportError:
        return "cpu"
DEVICE = _detect_device()
```
- Tự dùng GPU nếu có (Colab/máy CUDA), không thì CPU.
- Import torch "mềm": chưa cài torch app vẫn khởi động được.

## 8. Vì sao thu âm + encode WAV ở client?

- Backend `/api/record` chỉ cần **lưu file** (`shutil.copyfileobj`), không cần thư viện xử lý audio nặng → backend gọn nhẹ.
- Phần phức tạp (ghi micro, downsample, ghi header WAV) làm bằng JS trên trình duyệt (`wav-recorder.js`).
- Kết quả: WAV 16-bit PCM mono đúng định dạng pipeline fine-tune cần.

## 9. Lưu trữ

```
storage/
├── dataset/
│   ├── metadata.csv      # tự sinh lại sau mỗi record/delete
│   └── wavs/0000.wav...  # các đoạn đã thu (đặt tên theo sentence_id)
├── uploads/              # file giọng mẫu tải lên (ref_tmp.wav)
└── outputs/              # các wav kết quả sinh ra (uuid.wav)
```

## 10. Nguyên tắc thiết kế cốt lõi

1. **Backend mỏng, engine dày**: điều phối tách khỏi model; model pluggable.
2. **Lazy + soft import**: khởi động nhanh, chịu được khi thiếu thư viện nặng.
3. **Đẩy xử lý audio sang client**: backend không cần ffmpeg/librosa để nhận thu âm.
4. **Consent là chốt chặn**: `/api/tts` từ chối nếu `consent=False`.
5. **CPU chạy được, GPU nhanh hơn**: tự phát hiện device, khuyến nghị Colab cho fine-tune.
