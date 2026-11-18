# 4. Viết mục Dự án trong CV

Mục dự án là phần nhà tuyển dụng đọc kỹ nhất với fresher/junior (vì chưa có nhiều kinh nghiệm đi làm). Viết tốt = được gọi phỏng vấn.

## Cấu trúc mỗi dự án
```
[Tên dự án] — [một dòng nó là gì] · [Stack chính]
- Bullet 1: làm được gì + công nghệ + (số liệu nếu có)
- Bullet 2: một điểm kỹ thuật khó/nổi bật
- Bullet 3: chất lượng/quy trình (test, CI, bảo mật...)
[Link GitHub / demo]
```

## Nguyên tắc viết bullet
- **Bắt đầu bằng động từ mạnh**: Xây / Thiết kế / Tự cài đặt / Tối ưu / Tích hợp / Kiểm thử.
- **Có "cái gì" + "bằng gì" + "kết quả"**: "Tự cài đặt thuật toán TOTP theo RFC 6238 bằng thư viện chuẩn Python, kiểm chứng bằng test vector chính thức."
- **Có số liệu**: số dòng code, số test, số tính năng, hiệu năng.
- **Tránh chung chung**: thay "làm frontend" → "xây UI kéo-thả bằng JS thuần, không framework".

## Ví dụ mẫu (từ các dự án trong bộ)

**OTP Vault — Trình quản lý mã 2FA (TOTP) cục bộ · Python, FastAPI, cryptography**
- Tự cài đặt thuật toán **TOTP theo chuẩn RFC 6238** (HMAC + dynamic truncation) chỉ bằng thư viện chuẩn; kiểm chứng bằng **test vector chính thức** cho SHA1/256/512.
- Thiết kế mô hình **mã hóa "không lưu khóa"**: khóa suy từ mật khẩu chủ qua **scrypt**, mã hóa kho bằng **Fernet (AES)**, khóa chỉ tồn tại trong RAM.
- Viết bộ test gồm **property-based testing (hypothesis)** và test API qua FastAPI TestClient.
- GitHub: …

**Quán quen — Web app bản đồ quán ăn theo gu cá nhân · Vanilla JS, Leaflet, IndexedDB**
- Xây bản đồ tương tác (Leaflet + markercluster + heatmap), gợi ý "hôm nay đi đâu" theo mood với thuật toán chấm điểm có trọng số.
- Lưu trữ **hai tầng** (localStorage cho dữ liệu text, IndexedDB cho ảnh) để vượt giới hạn 5MB; backup **mã hóa AES-GCM**.
- **47 test Playwright** (E2E, mock fetch) + **CI/CD GitHub Actions** chỉ deploy khi test pass.

**Word Checker — Công cụ kiểm tra định dạng/chính tả file Word · Python, python-docx, Flask**
- Phân giải **"font hiệu lực"** qua 4 tầng kế thừa của OOXML để báo lỗi font chính xác.
- Kiến trúc tách lớp (mỗi loại kiểm tra một module, lỗi là `Issue` chung) dùng chung cho CLI lẫn web app; đóng gói thành `.exe` (PyInstaller).
- Chú trọng **giảm false positive** (bỏ qua công thức toán...) với bộ test riêng cho ca không-được-báo.

**Art Shield — Bảo vệ tranh khỏi AI bằng nhiễu đối kháng · Python, PyTorch, CLIP, FastAPI**
- Tự cài đặt thuật toán **PGD** và **EoT (Expectation over Transformation)** trên PyTorch để tạo nhiễu đối kháng làm hỏng đặc trưng CLIP của ảnh; nâng độ bền trước nén/resize từ **0/9 lên 9/9**.
- Thêm **perceptual masking** giấu nhiễu vào vùng nhiều chi tiết (SSIM 0.94 — gần vô hình) và mẹo **STE** cho gradient chảy qua phép lượng tử hóa.
- Tự đo **transferability** giữa các model (B/32 vs B/16) và trung thực về giới hạn; web app FastAPI kéo-thả với preset Nhẹ/Vừa/Mạnh.

**AI Usage Monitor — Đo token/credit dùng AI từ log local · Python, FastAPI, tiktoken**
- Xây bộ **quét log phòng thủ** (lọc regex, giới hạn 80MB, chống trùng fingerprint) dò object usage **đệ quy** qua nhiều schema (Claude/Kiro/Antigravity).
- Đếm token chính xác bằng **tiktoken** với ánh xạ model→encoding + `lru_cache`; phân biệt rõ token vs credit và **không bịa số liệu** khi nhà cung cấp chỉ lộ credit.
- Hoàn toàn **local, chỉ-đọc**, chống XSS phía frontend.

**Voice Clone Web — Clone giọng nói tiếng Việt (TTS) · Python, FastAPI, viXTTS, Web Audio API**
- Tích hợp **viXTTS** clone giọng zero-shot từ vài giây audio (conditioning latents); engine thiết kế **pluggable** + **lazy-load** + soft import.
- Tự viết **WAV encoder phía trình duyệt** (Web Audio → downsample → header 44 byte → int16) để backend chỉ việc lưu file.
- Pipeline **dataset → fine-tune** (200 câu phủ thanh điệu/âm vị, xuất LJSpeech, notebook Colab); đặt **bước xác nhận đồng ý bắt buộc** vì lý do đạo đức.

## Mẹo
- Sắp xếp dự án **mạnh nhất / liên quan JD nhất lên đầu**.
- Mỗi dự án 2–4 bullet là đủ; chất hơn lượng.
- Luôn kèm **link GitHub** (và demo nếu có) — nhà tuyển dụng sẽ bấm vào.
- Đảm bảo bạn **giải thích được mọi bullet** khi bị hỏi sâu — đừng viết thứ bạn không hiểu.

> 💡 Một bullet tốt là một "mồi câu hỏi": viết cái bạn MUỐN bị hỏi để khoe chiều sâu (vd "font hiệu lực", "không lưu khóa", "rolling summary").
