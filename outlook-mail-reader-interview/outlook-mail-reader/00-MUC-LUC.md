# 📚 Bộ tài liệu ôn phỏng vấn — Dự án Outlook Mail Reader

Chào bạn. Đây là bộ tài liệu giúp bạn **đi từ con số 0 đến tự tin trả lời phỏng vấn** về dự án Outlook Mail Reader (hộp thư đa tài khoản). Đọc theo thứ tự dưới đây.

> Toàn bộ viết sau khi **đọc trực tiếp 100% mã nguồn thật**: `config.py`, `main.py`, `store.py`, `engine.py` (1524 dòng), `routes.py`, `frontend/index.html`, `frontend/app.js` (810 dòng).

---

## 🗺️ Lộ trình học (đọc theo thứ tự)

| # | File | Nội dung | Đọc khi |
|---|------|----------|---------|
| 1 | `01-tong-quan-du-an.md` | Dự án là gì, giải quyết vấn đề gì, tech stack, cách chạy | Đầu tiên |
| 2 | `02-khai-niem-nen-tang.md` | Lý thuyết nền: OAuth 2.0, MSAL, device code flow, refresh token, Graph API, IMAP XOAUTH2, mã hóa Fernet, threading, DNS-over-HTTPS, iframe sandbox | Khi chưa rõ thuật ngữ |
| 3 | `03-kien-truc-he-thong.md` | Kiến trúc, luồng đăng nhập, 2 chế độ auth, luồng dữ liệu | Sau khi nắm khái niệm |
| 4 | `04-code-chi-tiet.md` | Giải thích từng file (config/main/store/engine/routes/frontend) | Học sâu phần code |
| 5 | `05-tinh-nang-noi-bat.md` | Phân tích sâu tính năng "ăn điểm" | Để kể chuyện ấn tượng |
| 6 | `06-cau-hoi-phong-van.md` | 70+ câu hỏi & câu trả lời mẫu | Luyện tập trả lời |
| 7 | `07-cheat-sheet.md` | Tóm tắt 1 trang xem trước giờ G | Phút chót |
| 8 | `08-code-walkthrough.md` | Phụ lục: đi qua TỪNG HÀM thật trong engine.py | Khi muốn hiểu 100% mã nguồn |

---

## ⏱️ Nếu bạn chỉ có ít thời gian

- **Còn 1 ngày**: file 1 → 2 → 3 → 6 → 7.
- **Còn 2 giờ**: file 1 → 3 → 7, rồi lướt 6.
- **Còn 30 phút**: file 7 + phần "Câu hỏi hay gặp nhất" trong file 6.

---

## 📦 Tóm tắt dự án trong 3 câu (học thuộc)

> Outlook Mail Reader là một **web app gộp hộp thư của nhiều tài khoản Outlook/Microsoft 365** vào một chỗ: mỗi tài khoản chỉ **đăng nhập OAuth một lần** (device code flow), sau đó app tự dùng **refresh token** (lưu **mã hóa** dưới đĩa) để lấy mail mà không cần đăng nhập lại — **không bao giờ lưu mật khẩu**. Backend là **FastAPI (Python)** dùng **MSAL + Microsoft Graph API** (và **IMAP XOAUTH2** làm dự phòng), hỗ trợ unified inbox, tìm kiếm, đánh dấu đã đọc, đính kèm, import refresh token hàng loạt bằng **job đa luồng có tiến độ/hủy**, và kiểm tra email bằng tra cứu **DNS/MX**. Frontend là **JS thuần**, render nội dung HTML của mail trong **iframe sandbox** an toàn.

Chúc bạn phỏng vấn thật tốt! 🍀
