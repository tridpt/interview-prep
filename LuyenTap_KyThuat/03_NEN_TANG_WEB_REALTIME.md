# 3. Nền tảng Web & Real-time

Phần ai cũng nên ôn — HTTP, REST, streaming, PWA, bảo mật web. Gắn với các dự án.

## 1. HTTP cơ bản
- **Request/Response**, method: GET (lấy), POST (tạo/gửi), PATCH (sửa một phần), PUT (thay cả), DELETE (xóa).
- **Status code**: 200 OK · 201 Created · 301/302 redirect · 400 bad request · 401 chưa xác thực · 403 cấm · 404 không thấy · 413 payload quá lớn · 429 quá nhiều request · 500 lỗi server · 502/503 upstream lỗi.
- **Header** (Content-Type, Authorization: Bearer ...), **body** (thường JSON).
- **Idempotent**: GET/PUT/DELETE lặp lại cho cùng kết quả; POST thì không.

## 2. REST API
- Tài nguyên qua URL (`/api/accounts`, `/api/accounts/{id}`), thao tác qua method.
- Stateless: mỗi request tự chứa đủ thông tin (vd agent-talk gửi cả lịch sử mỗi lượt).
- Các dự án trong bộ đều theo REST: totp-manager, outlook, word-checker (web).

## 3. Streaming & real-time
- **HTTP streaming / SSE**: server đẩy dần dữ liệu một chiều (agent-talk stream từng chữ LLM: dòng `data: {...}`).
- **WebSocket**: hai chiều, thời gian thực liên tục (khi nào cần? chat realtime, game). Streaming một chiều thì SSE/HTTP gọn hơn.
- **Polling**: client hỏi lại định kỳ (outlook poll trạng thái job import mỗi 1.5s; totp-manager load mã mỗi 1s).

## 4. CORS
- Cơ chế trình duyệt chặn gọi API khác origin trừ khi server cho phép (`Access-Control-Allow-Origin`).
- Các dự án local bật `allow_origins=["*"]` (an toàn vì chạy localhost).

## 5. PWA (Progressive Web App)
- **Manifest** + **Service Worker** + HTTPS → cài như app, chạy offline.
- **Vòng đời SW**: install (precache) → activate (dọn cache cũ) → fetch (chặn request).
- **Chiến lược cache**: cache-first (tile bản đồ), network-first (trang), stale-while-revalidate (asset). quan-quen-map dùng cả 4 theo origin; cap-cuu-101 precache app shell.
- **Cache versioning**: bump `CACHE_VERSION` để client nhận bản mới.
- Lưu ý bảo mật: totp-manager **không cache `/api`** (dữ liệu nhạy cảm + mã đổi mỗi giây).

## 6. Lưu trữ phía client
- **localStorage** (text, đồng bộ, ~5MB) vs **IndexedDB** (object/blob, async, lớn) vs **Cache API** (request/response, do SW quản).

## 7. Bảo mật web (hay hỏi)
- **XSS**: chèn mã qua nội dung người dùng → chống bằng escape khi nhét vào innerHTML (mọi dự án có `escapeHtml`).
- **CSRF**, **path traversal** (agent-talk chặn mã phiên bằng regex `[A-Za-z0-9]`), **giới hạn kích thước upload** (word-checker 25MB).
- **HTTPS/TLS**, **mã hóa-tại-chỗ** (totp-manager), **không lộ secret ra client** (chỉ trả `configured: true/false`).

## 8. OAuth 2.0 / xác thực (outlook-mail-reader)
- **Access token** (ngắn hạn) + **refresh token** (đổi lấy access mới); **device-code flow** (nhập mã trên trình duyệt); **scope** (quyền). Token xoay vòng → phải lưu lại token mới.

## Câu hỏi mẫu
- "SSE vs WebSocket?" → một chiều vs hai chiều.
- "Service worker cache thế nào để không kẹt bản cũ?" → version + network-first cho trang.
- "XSS là gì, chống sao?" → escape nội dung động.
- "401 vs 403?" → chưa xác thực vs đã xác thực nhưng không có quyền.
- "Vì sao không cache /api?" → dữ liệu động/nhạy cảm.

> 💡 Nối với dự án: "Em dùng network-first cho HTML để người dùng không kẹt bản cũ, còn tile bản đồ thì cache-first cho nhanh + xem offline."
