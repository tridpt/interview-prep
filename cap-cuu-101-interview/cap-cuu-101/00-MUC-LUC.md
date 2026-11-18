# 📚 Bộ tài liệu ôn phỏng vấn — Dự án Cấp Cứu 101

Chào bạn. Đây là bộ tài liệu giúp bạn **đi từ con số 0 đến tự tin trả lời phỏng vấn** về dự án Cấp Cứu 101 (Đừng Để Chết Nhảm). Mỗi file đi sâu một khía cạnh. Đọc theo thứ tự dưới đây.

---

## 🗺️ Lộ trình học (đọc theo thứ tự)

| # | File | Nội dung | Đọc khi |
|---|------|----------|---------|
| 1 | `01-tong-quan-du-an.md` | Dự án là gì, giải quyết vấn đề gì, demo thế nào | Đầu tiên, để hiểu bức tranh lớn |
| 2 | `02-khai-niem-nen-tang.md` | Lý thuyết nền: PWA, service worker, các Web API, i18n... | Khi chưa rõ thuật ngữ |
| 3 | `03-kien-truc-he-thong.md` | Kiến trúc, luồng khởi động, router, mô hình dữ liệu | Sau khi nắm khái niệm |
| 4 | `04-code-chi-tiet.md` | Giải thích từng file (app.js, sound.js, sw.js, index.html...) | Học sâu phần code |
| 5 | `05-tinh-nang-noi-bat.md` | Phân tích sâu các tính năng "ăn điểm" khi phỏng vấn | Để kể chuyện ấn tượng |
| 6 | `06-cau-hoi-phong-van.md` | 60+ câu hỏi & câu trả lời mẫu, từ dễ đến khó | Luyện tập trả lời |
| 7 | `07-cheat-sheet.md` | Tóm tắt 1 trang để xem lại ngay trước giờ G | Phút chót |
| 8 | `08-code-walkthrough.md` | Phụ lục: đi qua TỪNG HÀM thật trong app.js (2864 dòng) | Khi muốn hiểu cặn kẽ 100% mã nguồn |

---

## ⏱️ Nếu bạn chỉ có ít thời gian

- **Còn 1 ngày**: Đọc file 1 → 2 → 3 → 6 → 7.
- **Còn 2 giờ**: Đọc file 1 → 3 → 7, rồi lướt file 6.
- **Còn 30 phút**: Đọc file 7 (cheat sheet) và phần "Câu hỏi hay gặp nhất" trong file 6.

---

## 🎯 Cách dùng tài liệu hiệu quả

1. **Hiểu, đừng học vẹt.** Người phỏng vấn sẽ hỏi "tại sao", nên bạn cần hiểu lý do đằng sau mỗi quyết định.
2. **Tự kể lại dự án bằng lời của bạn** sau khi đọc file 1 và 3. Kể trôi chảy trong 2 phút là đạt.
3. **Trả lời to thành tiếng** các câu trong file 6.
4. **Chuẩn bị sẵn 2-3 câu chuyện**: thử thách khó nhất, quyết định kỹ thuật bạn tự hào, và cách bạn xử lý ràng buộc "app y tế nhưng vui".

---

## 📦 Tóm tắt dự án trong 3 câu (học thuộc)

> Cấp Cứu 101 là một **web app PWA** dạy kỹ năng sơ cứu sinh tử (CPR, hóc dị vật, bỏng, cầm máu...) qua các **mini-game hoạt hình hài hước** kiểu *Dumb Ways to Die*, kèm **Chế độ Hoảng loạn** đọc to từng bước khi gặp tình huống thật. Toàn bộ là **HTML/CSS/JS thuần — không framework, không build, không thư viện ngoài** — và **chạy offline 100%** nhờ service worker, điều cực kỳ quan trọng với một app cấp cứu. Dự án dùng nhiều Web API trình duyệt (Web Audio tự tổng hợp âm thanh, Web Speech để đọc to, Vibration, DeviceOrientation, Notifications, Web Share), hỗ trợ **song ngữ Việt/Anh** và lưu toàn bộ dữ liệu cục bộ bằng localStorage (không gửi đi đâu).

Chúc bạn phỏng vấn thật tốt! 🍀
