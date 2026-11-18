# 📚 Bộ tài liệu ôn phỏng vấn — Dự án Quán quen (Taste Map)

Chào bạn. Đây là bộ tài liệu giúp bạn **đi từ con số 0 đến tự tin trả lời phỏng vấn** về dự án Quán quen. Mỗi file đi sâu một khía cạnh. Đọc theo thứ tự dưới đây.

> Toàn bộ tài liệu được viết sau khi **đọc trực tiếp mã nguồn thật** (app.js 4718 dòng, i18n.js, sw.js, index.html, style.css, file test Playwright 47 ca). Không suy đoán.

---

## 🗺️ Lộ trình học (đọc theo thứ tự)

| # | File | Nội dung | Đọc khi |
|---|------|----------|---------|
| 1 | `01-tong-quan-du-an.md` | Dự án là gì, giải quyết vấn đề gì, tech stack, cách chạy | Đầu tiên |
| 2 | `02-khai-niem-nen-tang.md` | Lý thuyết nền: PWA, Leaflet, IndexedDB, Web Crypto, dịch vụ bản đồ, i18n, Playwright/CI | Khi chưa rõ thuật ngữ |
| 3 | `03-kien-truc-he-thong.md` | Kiến trúc, luồng khởi động, render pipeline, mô hình dữ liệu | Sau khi nắm khái niệm |
| 4 | `04-code-chi-tiet.md` | Giải thích từng file (app.js, i18n.js, sw.js, index.html, tests) | Học sâu phần code |
| 5 | `05-tinh-nang-noi-bat.md` | Phân tích sâu tính năng "ăn điểm" khi phỏng vấn | Để kể chuyện ấn tượng |
| 6 | `06-cau-hoi-phong-van.md` | 70+ câu hỏi & câu trả lời mẫu | Luyện tập trả lời |
| 7 | `07-cheat-sheet.md` | Tóm tắt 1 trang xem trước giờ G | Phút chót |
| 8 | `08-code-walkthrough.md` | Phụ lục: đi qua TỪNG HÀM thật trong app.js | Khi muốn hiểu 100% mã nguồn |

---

## ⏱️ Nếu bạn chỉ có ít thời gian

- **Còn 1 ngày**: file 1 → 2 → 3 → 6 → 7.
- **Còn 2 giờ**: file 1 → 3 → 7, rồi lướt 6.
- **Còn 30 phút**: file 7 + phần "Câu hỏi hay gặp nhất" trong file 6.

---

## 📦 Tóm tắt dự án trong 3 câu (học thuộc)

> Quán quen là một **web app PWA** giúp lưu và quản lý quán ăn/cafe **theo gu cá nhân** trên bản đồ — thứ Google Maps không làm được: chấm điểm theo tiêu chí riêng (ngon, yên tĩnh, ổ cắm, hợp date, hợp làm việc), gợi ý "hôm nay đi đâu" theo mood, lập lịch trình nhiều quán với tuyến đường thật, và khám phá quán mới quanh bạn từ OpenStreetMap. Toàn bộ là **HTML/CSS/JS thuần (không framework, không bundler)** dùng **Leaflet** cho bản đồ; **không có backend** — dữ liệu lưu cục bộ bằng **localStorage + IndexedDB** (ảnh), đồng bộ tùy chọn qua **GitHub Gist**, backup **mã hóa AES-GCM**. Dự án có **47 test Playwright tự động + CI/CD GitHub Actions** (chỉ deploy khi test pass), song ngữ Việt/Anh và chạy offline.

Chúc bạn phỏng vấn thật tốt! 🍀
