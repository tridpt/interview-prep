# 02 — Khái niệm nền tảng (từ con số 0)

---

## 1. TTS (Text-to-Speech) là gì?

Công nghệ biến **văn bản → giọng nói**. Đầu vào là chuỗi ký tự, đầu ra là tín hiệu âm thanh (sóng). TTS hiện đại dùng mạng nơ-ron để tạo giọng tự nhiên, gần giống người thật.

## 2. Voice cloning (clone giọng) là gì?

TTS thông thường đọc bằng một (vài) giọng cố định đã huấn luyện sẵn. **Voice cloning** cho phép đọc bằng **giọng của một người cụ thể**, học từ một mẫu âm thanh. Có 2 cấp độ:

- **Zero-shot cloning**: chỉ cần một mẫu **rất ngắn** (vài giây), model "bắt" được đặc trưng giọng ngay, không cần huấn luyện lại. XTTS làm được điều này.
- **Fine-tune cloning**: huấn luyện tiếp model trên **nhiều phút** giọng của người đó → giọng giống và ổn định hơn, nhưng tốn công/tài nguyên.

## 3. XTTS và viXTTS

- **XTTS-v2** (Coqui): model TTS đa ngôn ngữ, clone zero-shot, kiến trúc dựa trên một bộ **GPT-like** sinh token âm thanh + bộ giải mã thành sóng.
- **viXTTS**: bản fine-tune XTTS-v2 cho **tiếng Việt** (dataset viVoice), **mở rộng tokenizer** để xử lý tiếng Việt tốt hơn. Dự án dùng repo `capleaf/viXTTS`.

## 4. Conditioning latents & speaker embedding

Đây là khái niệm lõi của clone giọng trong XTTS. Khi đưa audio mẫu vào, model trích ra 2 thứ:
- **speaker embedding**: vector đặc trưng "chất giọng" (timbre) của người nói.
- **gpt_cond_latent** (conditioning latents): biểu diễn ngữ cảnh âm thanh để bộ sinh "bắt chước" phong cách nói.

Trong code: `model.get_conditioning_latents(audio_path=...)` trả về cặp `(gpt_cond_latent, speaker_embedding)`. Sau đó `model.inference(text, language, gpt_cond_latent, speaker_embedding, ...)` sinh ra sóng giọng đọc `text` theo giọng đó.

> 🔑 Ý tưởng: tách "nội dung cần đọc" (text) khỏi "chất giọng" (latents từ audio mẫu). Đổi audio mẫu = đổi giọng, không cần train lại.

## 5. Âm thanh số: sample rate, PCM, WAV

- **Sample rate** (tần số lấy mẫu): số mẫu/giây, vd 22050 Hz, 24000 Hz. Cao hơn = chi tiết hơn.
- **PCM** (Pulse-Code Modulation): cách lưu âm thanh "thô" — mỗi mẫu là một số (16-bit = số nguyên −32768..32767).
- **WAV**: định dạng file chứa PCM kèm một **header 44 byte** mô tả (sample rate, số kênh, bit/mẫu...).
- **Mono vs stereo**: 1 kênh vs 2 kênh. Dataset TTS thường dùng **mono**.

> Vì sao dự án dùng WAV chứ không webm? Vì pipeline fine-tune (LJSpeech-style) cần WAV thô, không nén.

## 6. Web Audio API & thu âm trên trình duyệt

Trình duyệt cho phép truy cập micro qua `navigator.mediaDevices.getUserMedia`. Để lấy dữ liệu âm thanh thô:
- Tạo `AudioContext`, nối micro vào một **ScriptProcessor** (hoặc AudioWorklet).
- Mỗi khung (buffer) trả về mảng `Float32Array` (giá trị −1..1).
- Gom các khung lại, **downsample** về sample rate đích, rồi **encode WAV** (ghi header + chuyển float → int16).

> ⚠️ Thu micro yêu cầu trang chạy trên `localhost` hoặc HTTPS — giới hạn bảo mật của trình duyệt.

## 7. Downsampling (giảm tần số lấy mẫu)

Micro thường thu ở 44100/48000 Hz, nhưng ta muốn 22050 Hz. **Downsample** là lấy trung bình/chọn lọc mẫu để giảm sample rate. Code dự án dùng cách lấy trung bình các mẫu trong mỗi "cửa sổ" tỉ lệ — đơn giản nhưng đủ dùng.

## 8. Fine-tune là gì?

**Fine-tune** = huấn luyện *tiếp* một model đã có sẵn trên dữ liệu mới (ở đây: giọng của bạn) để nó chuyên biệt hơn. Cần:
- **Dataset**: các cặp (file audio, transcript) — định dạng **LJSpeech**: `wavs/0001.wav|câu đã đọc|speaker1`.
- **Thời lượng**: 10–60 phút, mỗi đoạn 2–15 giây, thu sạch.
- **GPU**: train trên Colab (T4/L4/A100). Notebook có thể dùng **Whisper** để tự tạo transcript nếu bạn chỉ có audio.

## 9. Metadata LJSpeech

`metadata.csv` phân tách bằng dấu `|`, mỗi dòng: `đường_dẫn_wav | nội_dung_câu | tên_speaker`. Đây là định dạng chuẩn cho nhiều pipeline TTS. Dự án tự sinh lại file này từ các wav đã thu.

## 10. Lazy loading model

Model viXTTS nặng (vài trăm MB) và lâu nạp. **Lazy load**: chỉ nạp model vào bộ nhớ khi **gọi TTS lần đầu**, không nạp lúc khởi động app → server lên nhanh. Lần đầu sinh giọng sẽ lâu (nạp + có thể tải weights), các lần sau nhanh.

## 11. Engine pluggable (cắm-rút)

Thiết kế để dễ đổi model: định nghĩa một "giao diện" chung — bất kỳ engine nào có phương thức `synthesize(text, reference_wav, language) -> đường_dẫn_wav` đều dùng được. Muốn thử model khác chỉ cần viết class mới rồi gán `engine = YourEngine()`. Đây là một dạng **dependency inversion** đơn giản.

## 12. FastAPI: Form, File, UploadFile, StreamingResponse

- **`Form(...)`**: nhận dữ liệu dạng form (không phải JSON), hợp với upload file.
- **`UploadFile` / `File(...)`**: nhận file tải lên (audio).
- **`FileResponse`**: trả về một file (wav kết quả).
- **`StreamingResponse`**: trả dữ liệu theo luồng (vd zip dataset tạo trong bộ nhớ).

---

## Bảng thuật ngữ nhanh

| Thuật ngữ | Một câu |
|-----------|---------|
| TTS | Biến văn bản thành giọng nói |
| Voice cloning | Đọc bằng giọng học từ mẫu âm thanh |
| Zero-shot | Clone chỉ từ vài giây, không train lại |
| XTTS / viXTTS | Model TTS đa ngôn ngữ / bản fine-tune tiếng Việt |
| Conditioning latents | Đặc trưng giọng + ngữ cảnh trích từ audio mẫu |
| Speaker embedding | Vector "chất giọng" của người nói |
| Sample rate | Số mẫu âm thanh/giây (22050, 24000 Hz) |
| PCM / WAV | Âm thanh thô / file chứa PCM + header |
| Downsample | Giảm sample rate |
| Fine-tune | Huấn luyện tiếp trên dữ liệu mới |
| LJSpeech | Định dạng dataset (wav|text|speaker) |
| Lazy load | Chỉ nạp model khi cần |
| Pluggable engine | Đổi model dễ nhờ giao diện chung |
