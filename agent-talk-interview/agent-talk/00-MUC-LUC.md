# 📚 Bộ tài liệu ôn phỏng vấn — Dự án Agent Talk

Chào bạn. Đây là bộ tài liệu được viết để bạn **đi từ con số 0 đến tự tin trả lời phỏng vấn** về dự án Agent Talk. Mỗi file đi sâu vào một khía cạnh. Hãy đọc theo thứ tự dưới đây.

---

## 🗺️ Lộ trình học (đọc theo thứ tự)

| # | File | Nội dung | Đọc khi |
|---|------|----------|---------|
| 1 | `01-tong-quan-du-an.md` | Dự án là gì, giải quyết vấn đề gì, demo thế nào | Đầu tiên, để hiểu bức tranh lớn |
| 2 | `02-khai-niem-nen-tang.md` | Toàn bộ lý thuyết nền: LLM, API, streaming, async, FastAPI... | Khi chưa rõ thuật ngữ |
| 3 | `03-kien-truc-he-thong.md` | Kiến trúc tổng thể, luồng dữ liệu, sơ đồ | Sau khi nắm khái niệm |
| 4 | `04-backend-chi-tiet.md` | Giải thích từng file/endpoint backend | Học sâu phần server |
| 5 | `05-frontend-chi-tiet.md` | Giải thích logic frontend (vòng lặp hội thoại, lưu phiên...) | Học sâu phần client |
| 6 | `06-cac-tinh-nang-noi-bat.md` | Phân tích sâu các tính năng "ăn điểm" khi phỏng vấn | Để kể chuyện ấn tượng |
| 7 | `07-cau-hoi-phong-van.md` | 60+ câu hỏi & câu trả lời mẫu, từ dễ đến khó | Luyện tập trả lời |
| 8 | `08-cheat-sheet.md` | Tóm tắt 1 trang để xem lại ngay trước giờ G | Phút chót |
| 9 | `09-frontend-html-css.md` | Phụ lục: đọc trực tiếp index.html + style.css (mọi id/class) | Khi muốn hiểu cặn kẽ 100% mọi file |

---

## ⏱️ Nếu bạn chỉ có ít thời gian

- **Còn 1 ngày**: Đọc file 1 → 2 → 3 → 7 → 8.
- **Còn 2 giờ**: Đọc file 1 → 3 → 8, rồi lướt file 7.
- **Còn 30 phút**: Đọc file 8 (cheat sheet) và phần "Câu hỏi hay gặp nhất" trong file 7.

---

## 🎯 Cách dùng tài liệu hiệu quả

1. **Đọc hiểu, đừng học vẹt.** Người phỏng vấn sẽ hỏi vặn "tại sao", nên bạn cần hiểu lý do đằng sau mỗi quyết định.
2. **Tự kể lại dự án bằng lời của bạn** sau khi đọc xong file 1 và 3. Nếu kể trôi chảy được trong 2 phút là đạt.
3. **Tập trả lời to thành tiếng** các câu trong file 7. Trả lời trong đầu khác hẳn nói ra miệng.
4. **Chuẩn bị sẵn 2-3 câu chuyện** về "thử thách khó nhất" và "quyết định kỹ thuật bạn tự hào" (có gợi ý trong file 6 và 7).

---

## 📦 Tóm tắt dự án trong 3 câu (học thuộc câu này)

> Agent Talk là một ứng dụng web cho phép nhiều "nhân vật AI" có tính cách riêng tự trò chuyện, tranh biện với nhau theo thời gian thực, người dùng ngồi xem và có thể xen vào điều hướng. Backend dùng **FastAPI (Python)** đóng vai trò điều phối: gọi các mô hình ngôn ngữ lớn (LLM) qua API chuẩn OpenAI, hỗ trợ **streaming** từng chữ, tóm tắt khi hội thoại dài, tra cứu web, chấm điểm và bộ nhớ dài hạn. Frontend là **HTML/CSS/JavaScript thuần** (không framework), quản lý vòng lặp lượt nói, lưu phiên bằng localStorage và xuất file.

Chúc bạn phỏng vấn thật tốt! 🍀
