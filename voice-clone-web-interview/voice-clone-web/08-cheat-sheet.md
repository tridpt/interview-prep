# 08 — Cheat Sheet (lướt phút chót)

## Pitch 30 giây
> Voice Clone Web là web app clone giọng tiếng Việt bằng viXTTS. Thu/tải giọng mẫu vài giây, nhập text, model trích conditioning latents rồi đọc text bằng giọng đó. Backend FastAPI engine pluggable load lazy; thu âm tự encode WAV 16-bit ngay trên trình duyệt; có studio thu dataset + notebook fine-tune. Consent bắt buộc vì đạo đức.

## Ba điểm nhấn
1. **Clone qua conditioning latents** (speaker embedding + gpt_cond_latent) → zero-shot.
2. **WAV encoder tự viết ở client** (Web Audio → downsample → header 44 byte → int16).
3. **Engine pluggable + lazy-load + soft import**.

## Endpoint
| Method | Path | Vai trò |
|--------|------|---------|
| GET | `/api/health` | device + số mẫu đã thu |
| GET | `/api/sentences` | 200 câu + cờ recorded |
| POST | `/api/record` | lưu 1 đoạn thu |
| DELETE | `/api/record/{id}` | xóa để thu lại |
| GET | `/api/dataset/download` | zip dataset (BytesIO) |
| POST | `/api/tts` | sinh giọng (consent bắt buộc) |

## Luồng TTS
consent? → validate text (≤1000) → _resolve_reference (upload > id > mẫu đầu) → engine.synthesize → get_conditioning_latents → inference(temperature=0.3, text_splitting) → lưu uuid.wav (24kHz) → FileResponse.

## synthesize (nhớ nằm lòng)
`_load()` → `_normalize_vi()` → `get_conditioning_latents(audio mẫu)` → `inference(text, latents)` → `_save_wav(24000)`.

## Thuật ngữ 1 dòng
- **TTS**: văn bản → giọng nói.
- **Voice cloning**: đọc bằng giọng học từ mẫu.
- **Zero-shot**: clone từ vài giây, không train lại.
- **viXTTS**: XTTS-v2 fine-tune tiếng Việt.
- **Conditioning latents**: đặc trưng giọng + ngữ cảnh từ audio mẫu.
- **PCM/WAV**: âm thanh thô / file + header 44 byte.
- **Downsample**: giảm sample rate.
- **LJSpeech**: `wavs/x.wav|text|speaker`.
- **Lazy load**: nạp model khi cần.
- **Pluggable engine**: đổi model qua giao diện `synthesize`.

## Số liệu / tham số
- Sample rate thu: 22050 Hz; đầu ra viXTTS: 24000 Hz.
- MAX_TEXT_LENGTH = 1000; MIN_REFERENCE_SECONDS = 3.
- Fine-tune: 10–60 phút (15–30 đủ tốt), đoạn 2–15s.
- Repo: `capleaf/viXTTS` (Hugging Face).

## WAV header 44 byte
RIFF + size · WAVE · fmt (PCM=1, mono=1, sampleRate, byteRate=sr×2, blockAlign=2, bits=16) · data + size. Mẫu: float[-1,1] → int16 little-endian.

## Điểm "ăn điểm"
- WAV encoder tự viết (xử lý tín hiệu thật).
- Engine pluggable (decoupling), lazy + soft import.
- Pipeline dataset → fine-tune (200 câu phủ âm vị, Whisper transcript).
- Consent gate ở cả UI lẫn API.
- Streaming zip trong bộ nhớ.

## Câu "tủ" về đạo đức
> "Clone giọng dễ bị lạm dụng để mạo danh, nên tôi đặt bước xác nhận đồng ý bắt buộc ở cả UI lẫn API; /api/tts từ chối nếu thiếu consent. Dự án định vị rõ là học tập/nghiên cứu, chỉ dùng giọng của mình hoặc có đồng ý."

## Bẫy hay gặp
- Thu micro cần localhost/HTTPS.
- Mount `/` StaticFiles đặt **cuối**.
- Import nặng (TTS/torch/vinorm) phải "mềm" để app khởi động được.
- WAV chứ không webm (pipeline cần PCM thô).
- CPU chậm → khuyến nghị GPU/Colab.
