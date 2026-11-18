# 06 — Tính năng nổi bật (điểm "ăn điểm")

---

## 1. WAV encoder tự viết ngay trên trình duyệt ⭐⭐⭐

**Vì sao nổi bật**: thể hiện kiến thức **xử lý tín hiệu số** thực sự — hiểu cấu trúc file WAV và biết tại sao cần nó.

**Cách kể**: "Tôi thu micro qua Web Audio API, gom các khung Float32, downsample về 22050 Hz, rồi tự dựng header WAV 44 byte và chuyển từng mẫu float [-1,1] sang int16 little-endian. Dùng WAV chứ không webm vì pipeline fine-tune cần PCM thô. Nhờ vậy backend chỉ việc lưu file, không cần ffmpeg/librosa."

## 2. Engine pluggable (decoupling backend ↔ model) ⭐⭐⭐

**Vì sao nổi bật**: thiết kế hướng mở rộng, một dạng dependency inversion.

**Cách kể**: "Backend chỉ gọi `engine.synthesize(text, reference_wav)`, không biết bên trong là model gì. Muốn đổi sang VieNeu-TTS hay F5-TTS chỉ cần viết một class có cùng phương thức rồi gán lại `engine`. Tách rời này giúp thử nghiệm model mới mà không đụng backend."

## 3. Lazy-load + soft import (khởi động nhanh, chịu lỗi) ⭐⭐⭐

**Cách kể**: "Model viXTTS nặng vài trăm MB nên tôi chỉ nạp khi gọi TTS lần đầu — app lên ngay. Mọi import nặng (TTS, torch, vinorm) đều bọc try/except: thiếu thư viện thì app vẫn khởi động và báo lỗi rõ ràng đúng lúc cần, thay vì sập ngay."

## 4. Voice cloning qua conditioning latents ⭐⭐

**Cách kể**: "viXTTS tách 'nội dung' khỏi 'chất giọng'. Tôi gọi `get_conditioning_latents` trên audio mẫu để lấy speaker embedding + gpt_cond_latent, rồi `inference` sinh sóng đọc văn bản bằng giọng đó. Đổi audio mẫu là đổi giọng, không cần train lại — đó là zero-shot cloning."

## 5. Pipeline dataset → fine-tune hoàn chỉnh ⭐⭐

**Cách kể**: "Ngoài clone zero-shot, tôi có studio thu 200 câu mẫu thiết kế phủ đủ thanh điệu/âm vị, tự sinh metadata LJSpeech, đóng gói dataset.zip, kèm notebook Colab fine-tune (có dùng Whisper tạo transcript). Người dùng nâng chất lượng giọng từ zero-shot lên fine-tune."

## 6. 200 câu mẫu thiết kế có chủ đích ⭐

**Cách kể**: "Chất lượng fine-tune phụ thuộc dataset phủ âm vị. Tôi soạn 200 câu có đủ 6 thanh điệu, phụ âm khó (khúc khuỷu, ngoằn ngoèo), số/ngày tháng, câu hỏi/cảm thán — để giọng train ra tự nhiên và chắc."

## 7. Chuẩn hóa text tiếng Việt có fallback ⭐

**Cách kể**: "Tôi dùng vinorm để đọc số/ký hiệu tự nhiên (15% → mười lăm phần trăm), nhưng vinorm hay lỗi build trên Python 3.13 nên tôi fallback mềm: thiếu thì dùng text gốc, engine vẫn chạy."

## 8. Consent bắt buộc — đạo đức trong thiết kế ⭐⭐

**Cách kể**: "Clone giọng dễ bị lạm dụng (mạo danh, lừa đảo). Tôi đặt bước xác nhận đồng ý bắt buộc ở cả UI lẫn API: `/api/tts` từ chối nếu thiếu consent. Định vị rõ dự án là học tập/nghiên cứu, chỉ dùng với giọng của mình hoặc có đồng ý."

## 9. Đóng gói dataset trong bộ nhớ (streaming zip) ⭐

**Cách kể**: "Tôi nén dataset vào BytesIO rồi trả StreamingResponse — không tạo file zip tạm trên đĩa, sạch sẽ và nhanh."

---

## Bảng "kể gì theo loại JD"

| JD nhấn vào | Nhấn tính năng |
|-------------|----------------|
| AI / ML / Speech | viXTTS, conditioning latents, zero-shot vs fine-tune, dataset |
| Backend (Python) | FastAPI, engine pluggable, lazy/soft import, streaming zip, consent gate |
| Frontend / Web | WAV encoder tự viết, Web Audio API, quản lý thu âm, object URL |
| Xử lý tín hiệu/audio | WAV/PCM, downsample, sample rate, float→int16 |
| Đạo đức AI / trust&safety | consent bắt buộc, định vị mục đích, cảnh báo lạm dụng |
