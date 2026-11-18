# 📑 Mục lục — Bộ tài liệu phỏng vấn dự án Voice Clone Web (viXTTS)

Chào bạn! Bộ tài liệu này giúp bạn hiểu cặn kẽ dự án **Web Clone Giọng Nói Tiếng Việt** để đi phỏng vấn, kể cả khi bạn chưa biết gì về TTS hay xử lý âm thanh. Đọc theo thứ tự `00 → 08`.

> Đây là **web app** cho phép **thu âm / tải lên giọng mẫu** rồi **clone giọng để đọc văn bản tiếng Việt**. Engine mặc định là **viXTTS** (fine-tune từ XTTS-v2 cho tiếng Việt), clone giọng chỉ từ vài giây audio. Dự án còn có **studio thu dataset** và **notebook fine-tune** trên Colab.

## Thứ tự đọc

| File | Tên | Nội dung | Khi nào đọc |
|------|-----|----------|-------------|
| `00` | **Mục lục** (file này) | Bản đồ tài liệu | Đầu tiên |
| `01` | [Tổng quan dự án](01-tong-quan-du-an.md) | Dự án là gì, giải quyết gì, công nghệ chính, đạo đức | Bức tranh lớn |
| `02` | [Khái niệm nền tảng](02-khai-niem-nen-tang.md) | TTS, voice cloning, XTTS, embedding giọng, WAV/PCM, fine-tune — từ số 0 | Nền tảng lý thuyết |
| `03` | [Kiến trúc hệ thống](03-kien-truc-he-thong.md) | Backend/frontend, 2 tab, luồng dữ liệu, engine pluggable | Sau khái niệm |
| `04` | [Chi tiết backend](04-backend-chi-tiet.md) | `main.py` (API), `tts_engine.py` (viXTTS), `config.py`, `sentences.py` | Phần lõi |
| `05` | [Chi tiết frontend](05-frontend-chi-tiet.md) | `app.js`, `wav-recorder.js` (thu + encode WAV), 2 tab | Giao diện + thu âm |
| `06` | [Tính năng nổi bật](06-tinh-nang-noi-bat.md) | Điểm "ăn điểm": lazy load, engine pluggable, WAV encoder client, dataset/fine-tune | Chọn điểm nhấn |
| `07` | [Câu hỏi phỏng vấn](07-cau-hoi-phong-van.md) | 60+ câu hỏi kèm trả lời mẫu | Luyện trả lời |
| `08` | [Cheat sheet](08-cheat-sheet.md) | Tóm tắt 1 trang | Phút chót |

## Mục tiêu sau khi đọc xong

1. Kể được dự án trong 2 phút: nó làm gì, cho ai, công nghệ gì.
2. Giải thích được voice cloning zero-shot vs fine-tune, XTTS, conditioning latents.
3. Hiểu được vì sao thu âm encode WAV **ngay trên trình duyệt**, và pipeline dataset → fine-tune.
4. Nói được phần **đạo đức/pháp lý**: chỉ clone giọng của mình hoặc có đồng ý; app có bước consent bắt buộc.

## Một câu để nhớ

> "Voice Clone Web là web app clone giọng tiếng Việt bằng viXTTS: người dùng thu/tải giọng mẫu vài giây, nhập văn bản, model trích đặc trưng giọng (conditioning latents) rồi đọc văn bản bằng chính giọng đó. Engine thiết kế pluggable, model load lazy, audio được encode WAV ngay ở trình duyệt, và có cả studio thu dataset + notebook fine-tune. Có bước xác nhận đồng ý bắt buộc vì lý do đạo đức."
