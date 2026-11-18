# 01 — Tổng quan dự án

## 1. Dự án này là gì? (cho người chưa biết gì)

**Text-to-Speech (TTS)** là công nghệ biến văn bản thành giọng nói. **Voice cloning** (clone giọng) là TTS nâng cao: thay vì đọc bằng một giọng cố định, model học **đặc trưng giọng** của một người từ một mẫu âm thanh ngắn, rồi đọc văn bản bất kỳ bằng *chính giọng đó*.

**Voice Clone Web** là một **web app** làm điều này cho **tiếng Việt**:
1. Người dùng **thu âm** (qua micro) hoặc **tải lên** một mẫu giọng (vài giây).
2. Nhập một đoạn văn bản tiếng Việt.
3. Model đọc đoạn văn bản đó bằng giọng đã clone, trả về file WAV để nghe/tải.

Ngoài ra app còn có **Studio thu dataset**: đọc lần lượt các câu mẫu để tạo bộ dữ liệu, xuất `dataset.zip` đem đi **fine-tune** (huấn luyện tiếp) cho chất lượng cao hơn.

## 2. Nó giải quyết vấn đề gì?

| Vấn đề | Cách xử lý |
|--------|-----------|
| TTS tiếng Việt chất lượng thấp / giọng máy | Dùng viXTTS — fine-tune cho tiếng Việt từ XTTS-v2 |
| Clone giọng thường cần nhiều dữ liệu | XTTS clone **zero-shot** chỉ từ ~6 giây audio |
| Muốn giọng giống hơn nữa | Studio thu dataset + notebook fine-tune trên Colab |
| Thu âm trên web ra định dạng pipeline cần | Tự encode **WAV 16-bit PCM** ngay trên trình duyệt |
| Lạm dụng clone giọng người khác | Bước **xác nhận đồng ý (consent) bắt buộc** trước khi sinh |

## 3. Engine: viXTTS là gì?

- **XTTS-v2** (Coqui): model TTS đa ngôn ngữ, clone giọng zero-shot mạnh.
- **viXTTS** (`capleaf/viXTTS` trên Hugging Face): bản fine-tune XTTS-v2 trên dataset tiếng Việt (viVoice), mở rộng tokenizer tiếng Việt → đọc tiếng Việt tự nhiên hơn.
- Chạy qua thư viện **Coqui TTS** (fork `coqui-tts` được maintain).

## 4. Công nghệ chính

| Thành phần | Công nghệ | Vai trò |
|-----------|-----------|---------|
| Backend | **FastAPI** + `uvicorn` | API thu âm, dataset, TTS |
| Model TTS | **viXTTS** (Coqui TTS, PyTorch) | Clone giọng + sinh tiếng nói |
| Tải model | **huggingface_hub** | Tự tải weights viXTTS |
| Xử lý audio (lưu) | **torchaudio**, soundfile | Lưu WAV kết quả |
| Chuẩn hóa text VI | **vinorm** (tùy chọn, có fallback) | Đọc số/ký hiệu tự nhiên hơn |
| Frontend | HTML/CSS/JS thuần | 2 tab, thu âm, gọi API |
| Thu âm | **Web Audio API** (tự viết WAV encoder) | Ghi micro → WAV 16-bit PCM |
| Fine-tune | Notebook Colab (Whisper + train) | Huấn luyện giọng riêng |

## 5. Cấu trúc thư mục

```
voice-clone-web/
├── backend/
│   ├── main.py          # FastAPI: /api/sentences, /api/record, /api/dataset/download, /api/tts
│   ├── tts_engine.py    # Wrapper viXTTS (pluggable, lazy-load)
│   ├── sentences.py     # 200 câu mẫu tiếng Việt để thu dataset
│   └── config.py        # Đường dẫn, device, model, giới hạn
├── frontend/
│   ├── index.html       # 2 tab: Thu âm dữ liệu / Thử giọng clone
│   ├── app.js           # Quản lý tab, gọi API
│   ├── wav-recorder.js  # Thu âm + encode WAV phía trình duyệt
│   └── style.css
├── finetune/            # Notebook fine-tune viXTTS trên Colab
└── storage/             # dataset đã thu + audio sinh ra
```

## 6. Hai chức năng chính

1. **Thử giọng clone (zero-shot)**: thu/tải giọng mẫu → nhập text → nghe ngay. Không cần train.
2. **Thu dataset + fine-tune**: đọc các câu mẫu → tải `dataset.zip` → fine-tune trên Colab → đem checkpoint về dùng. Cho chất lượng cao hơn zero-shot.

## 7. Đạo đức & pháp lý (rất quan trọng)

Dự án **nhấn mạnh** điều này, và bạn nên chủ động nói khi phỏng vấn:
- Chỉ dùng với **giọng của chính bạn** hoặc giọng có **đồng ý rõ ràng**.
- Clone giọng người khác để mạo danh/lừa đảo là **vi phạm pháp luật**.
- App có **bước xác nhận đồng ý bắt buộc** (checkbox consent) ở cả tab thu âm lẫn tab sinh giọng; thiếu consent → API từ chối.
- Mục đích định vị rõ: **học tập / nghiên cứu**.

## 8. Điểm khác biệt đáng kể

- **Engine pluggable**: muốn đổi sang model khác (VieNeu-TTS, F5-TTS) chỉ cần viết class mới có cùng phương thức `synthesize()`.
- **Lazy-load model**: app khởi động nhanh; model nặng chỉ nạp khi gọi TTS lần đầu.
- **WAV encoder tự viết ở client**: thu micro rồi tự downsample + encode WAV 16-bit PCM mono (không dùng webm) vì pipeline fine-tune cần WAV.
- **200 câu mẫu được thiết kế kỹ**: phủ 6 thanh điệu, phụ âm khó, số/ngày tháng, câu hỏi/cảm thán — để giọng train ra tự nhiên.

## 9. Pitch 30 giây

> "Voice Clone Web là web app clone giọng tiếng Việt bằng viXTTS. Người dùng thu hoặc tải một mẫu giọng vài giây, nhập văn bản, model trích conditioning latents từ giọng mẫu rồi đọc văn bản bằng chính giọng đó. Backend FastAPI với engine pluggable load lazy; phần thu âm tự encode WAV 16-bit ngay trên trình duyệt; và có studio thu dataset + notebook fine-tune cho chất lượng cao hơn. Tôi đặt bước xác nhận đồng ý bắt buộc vì vấn đề đạo đức của clone giọng."
