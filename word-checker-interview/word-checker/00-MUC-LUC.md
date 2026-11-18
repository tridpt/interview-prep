# 📚 Bộ tài liệu ôn phỏng vấn — Dự án Word Checker

Chào bạn. Bộ tài liệu này giúp bạn **đi từ con số 0 đến tự tin trả lời phỏng vấn** về dự án Word Checker — công cụ kiểm tra chính tả & định dạng file Word (.docx) trước khi nộp. Mỗi file đi sâu một khía cạnh. Đọc theo thứ tự dưới đây.

> Toàn bộ tài liệu được viết sau khi **đọc trực tiếp mã nguồn thật**: `config.py`, `cli.py`, `webapp.py`, `launch.py`, và toàn bộ package `checker/` (document, formatting, textcheck, spelling, mathdetect, headings, limits, stats, learn_profile, llm_spelling, autofix, annotate, resources, report, runner, issues), cùng tests và conftest. Không suy đoán.

---

## 🗺️ Lộ trình học (đọc theo thứ tự)

| # | File | Nội dung | Đọc khi |
|---|------|----------|---------|
| 1 | `01-tong-quan-du-an.md` | Dự án là gì, giải quyết vấn đề gì, tech stack, cách dùng | Đầu tiên |
| 2 | `02-khai-niem-nen-tang.md` | Lý thuyết nền: .docx/OOXML, python-docx, font hiệu lực, regex, profile, nhận diện toán, LLM, PyInstaller, pytest | Khi chưa rõ thuật ngữ |
| 3 | `03-kien-truc-he-thong.md` | Kiến trúc, luồng kiểm tra, mô hình dữ liệu Issue, runner dùng chung | Sau khi nắm khái niệm |
| 4 | `04-checker-chi-tiet.md` | Mổ xẻ từng module trong `checker/`, từng hàm | Học sâu phần lõi |
| 5 | `05-giao-dien-chi-tiet.md` | CLI, web app (Flask), autofix/annotate, đóng gói .exe | Học sâu phần giao diện & xuất kết quả |
| 6 | `06-tinh-nang-noi-bat.md` | Phân tích sâu tính năng "ăn điểm" | Để kể chuyện ấn tượng |
| 7 | `07-cau-hoi-phong-van.md` | 70+ câu hỏi & câu trả lời mẫu | Luyện tập trả lời |
| 8 | `08-cheat-sheet.md` | Tóm tắt 1 trang xem trước giờ G | Phút chót |

---

## ⏱️ Nếu bạn chỉ có ít thời gian

- **Còn 1 ngày**: file 1 → 2 → 3 → 7 → 8.
- **Còn 2 giờ**: file 1 → 3 → 8, rồi lướt 7.
- **Còn 30 phút**: file 8 + phần "Câu hỏi hay gặp nhất" trong file 7.

---

## 📦 Tóm tắt dự án trong 3 câu (học thuộc)

> Word Checker là một công cụ **chạy offline** rà soát file `.docx` trước khi nộp — kiểm tra **định dạng** (font, cỡ chữ, giãn dòng, canh lề, lề trang), **lỗi cơ học văn bản** (cách đôi, dấu cách thừa/thiếu, lặp từ, viết hoa đầu câu...), **chính tả tiếng Việt** (từ điển lỗi thường gặp + tùy chọn AI/LLM), **tiêu đề & đánh số mục**, và **giới hạn số từ/trang**. Lõi là **Python + python-docx** với một package `checker/` tách lớp sạch sẽ (mỗi loại kiểm tra một module, mọi lỗi là một `Issue` chung); dùng được qua **CLI** hoặc **web app kéo-thả (Flask)**, có **tự động sửa (--fix)**, **chú thích comment vào file (--annotate)**, xuất **HTML/PDF/JSON**, **học quy chuẩn từ file mẫu (--template)**, và **đóng gói thành .exe** (PyInstaller) chạy không cần cài Python. Dự án có **bộ test pytest đầy đủ** và CI GitHub Actions.

Chúc bạn phỏng vấn thật tốt! 🍀
