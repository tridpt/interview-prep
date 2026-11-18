# 📑 Mục lục — Bộ tài liệu phỏng vấn dự án Token Counter (AI Usage Monitor)

Chào bạn! Bộ tài liệu này giúp bạn hiểu cặn kẽ dự án **AI Usage Monitor** (tên thư mục: `token-counter`) để đi phỏng vấn, kể cả khi bạn chưa biết gì về token hay tokenizer. Đọc theo thứ tự `00 → 08`.

> Đây là **web app chạy local** có 2 chế độ: (1) **quét log local** của các công cụ lập trình AI (Claude Code, Kiro, Google Antigravity) để lấy usage thực tế khi có metadata; (2) **ước tính token** cho văn bản bạn dán vào, dùng thư viện `tiktoken`.

## Thứ tự đọc

| File | Tên | Nội dung | Khi nào đọc |
|------|-----|----------|-------------|
| `00` | **Mục lục** (file này) | Bản đồ tài liệu | Đầu tiên |
| `01` | [Tổng quan dự án](01-tong-quan-du-an.md) | Dự án là gì, giải quyết gì, công nghệ chính | Bức tranh lớn |
| `02` | [Khái niệm nền tảng](02-khai-niem-nen-tang.md) | Token, tokenizer, BPE, encoding, FastAPI, async — từ số 0 | Nền tảng lý thuyết |
| `03` | [Kiến trúc hệ thống](03-kien-truc-he-thong.md) | Backend/frontend, luồng dữ liệu, các API | Sau khái niệm |
| `04` | [Chi tiết backend](04-backend-chi-tiet.md) | `main.py` (đếm token) + `usage_scanner.py` (quét log) từng hàm | Phần lõi |
| `05` | [Chi tiết frontend](05-frontend-chi-tiet.md) | `app.js`, `index.html`, parse JSON usage, debounce | Giao diện |
| `06` | [Tính năng nổi bật](06-tinh-nang-noi-bat.md) | Điểm "ăn điểm": quét an toàn, dò usage đệ quy, trung thực dữ liệu | Chọn điểm nhấn |
| `07` | [Câu hỏi phỏng vấn](07-cau-hoi-phong-van.md) | 60+ câu hỏi kèm trả lời mẫu | Luyện trả lời |
| `08` | [Cheat sheet](08-cheat-sheet.md) | Tóm tắt 1 trang | Phút chót |

## Mục tiêu sau khi đọc xong

1. Kể được dự án trong 2 phút: nó làm gì, cho ai, công nghệ gì.
2. Giải thích được token/tokenizer/BPE và vì sao đếm token quan trọng (chi phí API).
3. Hiểu cơ chế **quét log an toàn**: lọc theo pattern, giới hạn dung lượng file, chống trùng (fingerprint), dò object usage đệ quy với nhiều tên khóa khác nhau.
4. Nói được triết lý **trung thực dữ liệu**: app không "bịa" token nếu nhà cung cấp chỉ lộ credit/quota.

## Một câu để nhớ

> "AI Usage Monitor là web app local đọc log của các công cụ AI để tổng hợp usage thật khi có metadata, đồng thời ước tính token cho văn bản bằng `tiktoken`. Điểm cốt lõi là **quét an toàn, chỉ-đọc, và trung thực** — không giả vờ có token chính xác khi nhà cung cấp chỉ lộ credit."
