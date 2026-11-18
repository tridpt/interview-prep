# 02 — Khái niệm nền tảng

## Prompt injection và ranh giới tin cậy

Prompt injection là cố biến nội dung ưu tiên thấp (tin nhắn hoặc tài liệu truy xuất) thành chỉ thị ưu tiên cao. Trong 5 màn RAG, tài liệu là **dữ liệu không đáng tin**; không được cho nó quyền sửa chính sách của trợ lý hoặc đọc bí mật.

## Mô hình phòng thủ trong game

- **Input filter**: blocklist từ khóa; dễ bỏ sót cách diễn đạt khác.
- **Output filter/scan**: tìm bí mật nguyên văn; các dạng mã hóa/biến đổi có thể né phép so chuỗi trong mô phỏng.
- **System strength/judge**: tăng ngưỡng hoặc chặn một số kỹ thuật đã phát hiện; không phải LLM judge thật ở chế độ mô phỏng.
- **Provenance/trust boundary**: coi mệnh lệnh trong tài liệu là dữ liệu; trong code đây vẫn là luật mô phỏng, không phải chứng minh tuyệt đối.

## Tất định so với LLM thật

`runAttack` và `runIndirect` dùng mẫu regex, trọng số và ngưỡng để tạo kết quả lặp lại được. LLM thật có độ bất định và chính sách khác; sandbox tùy chọn gọi provider qua `llm.js`, không dùng kết quả demo như số đo bảo mật sản phẩm thật.
