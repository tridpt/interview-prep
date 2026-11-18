# 07 — Câu hỏi phỏng vấn (kèm trả lời mẫu)

---

## A. Tổng quan (1–10)

**1. Dự án làm gì trong một câu?**
Web app clone giọng tiếng Việt: thu/tải giọng mẫu vài giây rồi đọc văn bản bất kỳ bằng chính giọng đó, dùng viXTTS.

**2. Hai chức năng chính?**
(1) Thử giọng clone zero-shot; (2) Studio thu dataset + fine-tune cho chất lượng cao hơn.

**3. Engine là gì?**
viXTTS — bản fine-tune XTTS-v2 cho tiếng Việt, chạy qua Coqui TTS.

**4. Công nghệ chính?**
FastAPI, viXTTS (Coqui TTS/PyTorch), huggingface_hub, torchaudio, vinorm (tùy chọn); frontend JS thuần + Web Audio API.

**5. Ai dùng?**
Người muốn tạo giọng đọc tiếng Việt bằng giọng của chính mình — học tập/nghiên cứu.

**6. Vì sao chọn viXTTS?**
Clone zero-shot chỉ từ ~6 giây, và đã fine-tune cho tiếng Việt nên đọc tự nhiên hơn XTTS gốc.

**7. Các endpoint chính?**
`/api/sentences`, `/api/record` (POST/DELETE), `/api/dataset/download`, `/api/tts`, `/api/health`.

**8. Đầu vào/đầu ra của TTS?**
Vào: text + giọng mẫu (file hoặc mẫu đã thu) + consent. Ra: file WAV đọc text bằng giọng clone.

**9. Vấn đề đạo đức?**
Clone giọng dễ bị lạm dụng; app có consent bắt buộc, chỉ dùng giọng của mình hoặc có đồng ý.

**10. Hạn chế?**
Zero-shot chưa giống hoàn hảo; chạy CPU chậm (cần GPU/Colab); chưa có auth nếu deploy công khai.

---

## B. Khái niệm nền tảng (11–28)

**11. TTS là gì?**
Công nghệ biến văn bản thành giọng nói.

**12. Voice cloning khác TTS thường thế nào?**
TTS thường đọc bằng giọng cố định; cloning đọc bằng giọng học từ mẫu âm thanh của một người.

**13. Zero-shot cloning là gì?**
Clone giọng chỉ từ vài giây audio, không cần huấn luyện lại.

**14. Fine-tune cloning khác gì?**
Huấn luyện tiếp trên nhiều phút giọng → giống và ổn định hơn, nhưng tốn tài nguyên.

**15. XTTS-v2 và viXTTS?**
XTTS-v2: model TTS đa ngôn ngữ clone zero-shot; viXTTS: bản fine-tune tiếng Việt, mở rộng tokenizer.

**16. Conditioning latents là gì?**
Đặc trưng trích từ audio mẫu để model "bắt chước" giọng: gồm speaker embedding + gpt_cond_latent.

**17. Speaker embedding là gì?**
Vector mô tả "chất giọng" (timbre) của người nói.

**18. Sample rate là gì?**
Số mẫu âm thanh lấy mỗi giây (vd 22050, 24000 Hz).

**19. PCM và WAV?**
PCM: cách lưu âm thanh thô (mỗi mẫu là số); WAV: file chứa PCM + header 44 byte mô tả.

**20. Vì sao dùng WAV chứ không webm?**
Pipeline fine-tune cần PCM thô không nén; WAV là chuẩn cho dataset TTS.

**21. Downsample là gì? Vì sao cần?**
Giảm sample rate (vd 48000 → 22050) bằng lấy trung bình mẫu; vì micro thu cao hơn mức ta cần.

**22. Fine-tune là gì?**
Huấn luyện tiếp một model có sẵn trên dữ liệu mới (giọng của bạn) để chuyên biệt hơn.

**23. Định dạng LJSpeech?**
`metadata.csv` phân tách `|`: `wavs/0001.wav|nội dung|speaker1`.

**24. Lazy load là gì? Lợi ích?**
Chỉ nạp model khi gọi lần đầu → app khởi động nhanh.

**25. Engine pluggable là gì?**
Thiết kế cho phép đổi model dễ nhờ giao diện chung (`synthesize`).

**26. Web Audio API dùng để làm gì ở đây?**
Truy cập micro, lấy dữ liệu âm thanh thô (Float32) để encode WAV.

**27. temperature trong inference là gì?**
Độ "ngẫu hứng" khi sinh; thấp (0.3) → giọng ổn định, rõ ràng.

**28. enable_text_splitting làm gì?**
Tự tách câu dài thành đoạn để đọc ổn định hơn.

---

## C. Backend chi tiết (29–46)

**29. Vì sao backend chỉ lưu file ở /api/record?**
Vì audio đã encode WAV ở client; backend không cần thư viện audio nặng.

**30. _rebuild_metadata làm gì?**
Sinh lại metadata.csv (LJSpeech) từ các wav hiện có để luôn đồng bộ.

**31. Đặt tên file thu theo gì?**
Theo sentence_id 4 chữ số (`0001.wav`) → thu lại sẽ ghi đè.

**32. /api/dataset/download tạo zip thế nào?**
Nén vào BytesIO trong bộ nhớ rồi StreamingResponse — không tạo file tạm trên đĩa.

**33. /api/tts validate gì trước khi sinh?**
Consent (bắt buộc), text không rỗng, text ≤ MAX_TEXT_LENGTH (1000).

**34. _resolve_reference ưu tiên thế nào?**
File upload > reference_id > mẫu thu đầu tiên; không có → lỗi 400.

**35. Vì sao import engine bên trong /api/tts?**
Lazy — để app không nặng lúc khởi động khi chưa cần TTS.

**36. Phân biệt lỗi RuntimeError vs Exception ở /api/tts?**
RuntimeError (chưa cài TTS) → 503; lỗi khác → 500, kèm thông báo rõ.

**37. _ensure_downloaded làm gì?**
Kiểm tra weights; thiếu thì snapshot_download từ Hugging Face.

**38. Vì sao import TTS bên trong _load?**
Để app vẫn khởi động khi chưa cài thư viện TTS; báo lỗi rõ khi thực sự cần.

**39. synthesize gồm các bước nào?**
load → normalize text → get_conditioning_latents(audio mẫu) → inference(text, latents) → lưu WAV uuid.

**40. Vì sao lưu kết quả bằng uuid?**
Tránh trùng tên file giữa các lần sinh.

**41. _save_wav lưu ở sample rate nào?**
24000 Hz (đầu ra của viXTTS), bằng torchaudio.

**42. _normalize_vi và fallback?**
Dùng vinorm chuẩn hóa số/ký hiệu; thiếu vinorm thì dùng text gốc (fallback mềm).

**43. _detect_device hoạt động ra sao?**
Thử import torch, trả "cuda" nếu có GPU, không thì "cpu"; thiếu torch cũng trả "cpu".

**44. config.py tự làm gì khi import?**
Tạo các thư mục model/uploads/outputs, đặt device, hằng giới hạn.

**45. sentences.py có gì đặc biệt?**
200 câu phủ thanh điệu/âm vị/số/câu hỏi — quan trọng cho chất lượng fine-tune.

**46. Mount tĩnh gồm gì?**
`/storage` (nghe lại audio) và `/` (frontend) — `/` đặt cuối.

---

## D. Frontend & câu hỏi sâu (47–62)

**47. WavRecorder encode WAV thế nào?**
Gom Float32 → downsample → dựng header WAV 44 byte → chuyển float [-1,1] sang int16 little-endian → Blob.

**48. Vì sao dùng ScriptProcessor?**
Đơn giản, tương thích rộng để gom buffer âm thanh (dù AudioWorklet hiện đại hơn).

**49. Header WAV 44 byte gồm gì?**
RIFF/WAVE, chunk fmt (PCM=1, mono=1, sampleRate, byteRate, bitsPerSample=16), chunk data + kích thước.

**50. Chuyển float sang int16 thế nào?**
`s < 0 ? s*0x8000 : s*0x7fff` (clamp [-1,1] trước) — ánh xạ về dải số nguyên 16-bit.

**51. Vì sao chỉ thu 1 câu một lúc?**
Tránh xung đột nhiều recorder; logic `toggleRecord` chặn nếu đang thu câu khác.

**52. Kết quả TTS được phát thế nào?**
`URL.createObjectURL(blob)` gán vào `<audio>` và nút tải.

**53. Consent được kiểm ở đâu?**
Cả frontend (alert/nút disabled) lẫn backend (/api/tts từ chối nếu thiếu).

**54. Vì sao chạy CPU chậm?**
Model TTS lớn; sinh sóng tốn tính toán. GPU (Colab) nhanh hơn nhiều lần.

**55. Lần sinh đầu tiên vì sao lâu?**
Phải nạp model (lazy) và có thể tải weights từ Hugging Face.

**56. Làm sao đổi sang model TTS khác?**
Viết class mới có `synthesize(text, reference_wav, language)` rồi gán `engine = YourEngine()`.

**57. Rủi ro bảo mật khi deploy công khai?**
Chưa có auth; API mở. README ghi rõ cần thêm auth nếu deploy thật.

**58. Cải tiến tiếp theo?**
Lọc nhiễu audio mẫu tự động, tách câu dài tốt hơn, thêm auth, hàng đợi job cho nhiều người dùng.

**59. Vì sao tách nội dung khỏi chất giọng?**
Để clone zero-shot: đổi audio mẫu là đổi giọng mà không train lại.

**60. Dataset cần bao nhiêu để fine-tune tốt?**
10–60 phút (15–30 phút đã tốt), mỗi đoạn 2–15 giây, thu sạch, ≥22kHz.

**61. Whisper dùng làm gì trong fine-tune?**
Tự tạo transcript từ audio nếu bạn chỉ có file âm thanh chưa có lời.

**62. Bài học lớn nhất từ dự án?**
Tách rời (pluggable/lazy) giúp tích hợp model nặng gọn gàng; và clone giọng phải đi kèm trách nhiệm đạo đức.

---

## Mẹo trả lời
- Nhấn 3 điểm: **clone qua conditioning latents**, **WAV encoder tự viết ở client**, **engine pluggable + lazy**.
- Luôn chủ động nhắc **đạo đức/consent** — người phỏng vấn đánh giá cao ý thức trách nhiệm.
- Trung thực về hạn chế (CPU chậm, chưa auth) rồi nêu hướng cải thiện.
