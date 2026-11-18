# 📚 Bộ tài liệu ôn phỏng vấn — Dự án AI Explorer

Chào bạn. Đây là bộ tài liệu giúp bạn **đi từ con số 0 đến tự tin trả lời phỏng vấn** về dự án AI Explorer — "bảo tàng AI tương tác" chạy hoàn toàn trên trình duyệt. Mỗi file đi sâu một khía cạnh. Đọc theo thứ tự dưới đây.

---

## 🗺️ Lộ trình học (đọc theo thứ tự)

| # | File | Nội dung | Đọc khi |
|---|------|----------|---------|
| 1 | `01-tong-quan-du-an.md` | Dự án là gì, giải quyết vấn đề gì, demo thế nào | Đầu tiên, để hiểu bức tranh lớn |
| 2 | `02-khai-niem-nen-tang.md` | Lý thuyết nền: ES Modules, PWA, service worker, i18n… **và** mọi khái niệm AI mà các phòng minh hoạ | Khi chưa rõ thuật ngữ |
| 3 | `03-kien-truc-he-thong.md` | Kiến trúc, router hashchange, mảng ROOMS, lộ trình học, cách thêm phòng mới | Sau khi nắm khái niệm |
| 4 | `04-code-chi-tiet.md` | Giải thích từng file (app.js, i18n.js, store.js, roomstate.js, roomquiz.js, sw.js, pwa.js, sound.js…) | Học sâu phần code |
| 5 | `05-tinh-nang-noi-bat.md` | Phân tích sâu các tính năng "ăn điểm" khi phỏng vấn | Để kể chuyện ấn tượng |
| 6 | `06-cau-hoi-phong-van.md` | 40+ câu hỏi & câu trả lời mẫu, từ dễ đến khó | Luyện tập trả lời |
| 7 | `07-cheat-sheet.md` | Tóm tắt 1 trang để xem lại ngay trước giờ G | Phút chót |
| 8 | `08-code-walkthrough.md` | Phụ lục: đi qua luồng chạy thật của `route()`, một file phòng điển hình, và phòng Reasoning | Khi muốn hiểu cặn kẽ mã nguồn |

---

## ⏱️ Nếu bạn chỉ có ít thời gian

- **Còn 1 ngày**: Đọc file 1 → 2 → 3 → 6 → 7.
- **Còn 2 giờ**: Đọc file 1 → 3 → 7, rồi lướt file 6.
- **Còn 30 phút**: Đọc file 7 (cheat sheet) và phần "5 câu hay gặp nhất" trong file 6.

---

## 🎯 Cách dùng tài liệu hiệu quả

1. **Hiểu, đừng học vẹt.** Người phỏng vấn sẽ hỏi "tại sao", nên bạn cần hiểu lý do đằng sau mỗi quyết định (vì sao không framework, vì sao ES modules, vì sao cache versioning…).
2. **Tự kể lại dự án bằng lời của bạn** sau khi đọc file 1 và 3. Kể trôi chảy trong 2 phút là đạt.
3. **Trả lời to thành tiếng** các câu trong file 6.
4. **Chuẩn bị sẵn 2-3 câu chuyện**: thử thách kỹ thuật khó nhất (vd tự tính tương phản WCAG cho chữ, hay bẫy focus cho ô tìm kiếm), một quyết định kiến trúc bạn tự hào (kiến trúc module hoá phòng), và **phòng Reasoning mới** (chain-of-thought) như một ví dụ bắt kịp xu hướng AI.

---

## 📦 Tóm tắt dự án trong 3 câu (học thuộc)

> AI Explorer là một **"bảo tàng AI tương tác"** gồm **26 phòng**, mỗi phòng giải thích trực quan một khái niệm AI (từ mạng nơ-ron, token, embeddings, attention… tới RAG, agents, RLHF và cả **chain-of-thought / reasoning models**) và cho người dùng **tự tay nghịch**. Toàn bộ chạy **100% trên trình duyệt bằng vanilla JS + ES Modules — không backend, không API key** — cài được như app và **chạy offline** nhờ service worker (PWA), song ngữ **Việt/Anh** toàn diện. Kiến trúc **module hoá**: mỗi phòng là một file độc lập export `render(root)`, thêm vào mảng `ROOMS` trong `app.js` là xong; CI chạy 3 lớp test (toàn vẹn cache, toàn vẹn song ngữ, smoke test Playwright) và **chặn deploy nếu test fail**.

Chúc bạn phỏng vấn thật tốt! 🍀
