# 06 — Tính năng nổi bật (điểm "ăn điểm")

---

## 1. Quét log "phòng thủ" trước dữ liệu thực tế bẩn ⭐⭐⭐

**Vì sao nổi bật**: log thật thì to, lộn xộn, có thể hỏng. Scanner xử lý mọi tình huống mà không crash.

**Cách kể**: "Trước khi parse JSON tốn kém, tôi lọc dòng bằng regex; bỏ qua file lớn hơn 80MB; đọc với `errors='ignore'`; bắt OSError/UnicodeError để file lỗi không làm sập cả lần quét. Mỗi bản ghi có fingerprint để chống đếm trùng."

## 2. Dò object usage đệ quy qua nhiều schema ⭐⭐⭐

**Vì sao nổi bật**: mỗi công cụ AI (Claude, Kiro, Gemini/Antigravity) ghi JSON cấu trúc khác nhau, tên khóa khác nhau. Một thuật toán phủ được tất cả.

**Cách kể**: "Tôi duyệt đệ quy toàn cây JSON, coi một dict là 'usage' nếu nó chứa bất kỳ khóa nào trong tập tên đã biết — `input_tokens`, `prompt_tokens`, `promptTokenCount`, `candidatesTokenCount`... Rồi tôi chuẩn hóa về một bản ghi chung bằng cách thử lần lượt nhiều tên khóa."

## 3. Triết lý "trung thực dữ liệu" ⭐⭐⭐

**Vì sao nổi bật**: rất hiếm và rất "chín". App không bịa số liệu.

**Cách kể**: "Nếu một nhà cung cấp chỉ lộ credit và % context mà không ghi token (như Kiro trên máy thử), tôi báo đúng credit kèm ghi chú 'không có token split', chứ không suy ra token giả. Tôi cũng nói rõ tiktoken là tokenizer OpenAI nên với Claude/Gemini chỉ là ước tính."

## 4. Đếm token đúng encoding + cache ⭐⭐

**Cách kể**: "Tôi ánh xạ model → encoding (`o200k_base`, `cl100k_base`) với fallback 3 tầng, và bọc `lru_cache` để không tạo lại encoder mỗi lần gõ — quan trọng vì ô đếm gọi liên tục theo debounce."

## 5. Hoàn toàn local, chỉ-đọc (privacy-first) ⭐⭐

**Cách kể**: "Scanner chỉ đọc, không bao giờ ghi/sửa/xóa, và không gửi gì ra mạng. Toàn bộ chạy trên máy người dùng. Đây là cam kết quyền riêng tư cốt lõi."

## 6. Hai công cụ tách biệt, chịu lỗi độc lập ⭐

**Cách kể**: "Import tiktoken kiểu mềm: thiếu thì phần đếm báo lỗi rõ qua `/api/health`, nhưng phần quét log vẫn chạy. Hai mảng không kéo nhau sập."

## 7. Chống XSS ở frontend ⭐

**Cách kể**: "Dữ liệu log là không tin cậy, nên tôi `escapeHtml` mọi giá trị trước khi nhét vào innerHTML — tránh chèn mã độc qua đường dẫn/model bất thường trong log."

## 8. Parse usage chạy được cả client lẫn server ⭐

**Cách kể**: "Ô 'dán JSON usage' parse ngay trên trình duyệt bằng cùng logic đệ quy như backend, cho phản hồi tức thì mà không cần round-trip."

---

## Bảng "kể gì theo loại JD"

| JD nhấn vào | Nhấn tính năng |
|-------------|----------------|
| Backend / API (Python) | FastAPI, Pydantic, lru_cache, import mềm, endpoint thiết kế gọn |
| Data / log processing | quét phòng thủ, dò usage đệ quy, chuẩn hóa schema, fingerprint |
| AI / LLM tooling | token/tokenizer/BPE, tiktoken, input/output/cache/reasoning tokens |
| Bảo mật / privacy | local-only chỉ-đọc, escapeHtml, không gửi mạng |
| Frontend | DOM thuần, debounce, render giới hạn, trạng thái UI |
