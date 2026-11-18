# 01 — Tổng quan dự án

**Pitch 30 giây:** Red-Team Lab là game giáo dục bảo mật chạy trên trình duyệt, dạy prompt injection và cách phòng thủ bằng **bộ mô phỏng quy tắc tất định**. Có 7 màn chat trực tiếp, 5 màn tài liệu độc/RAG, chế độ sandbox tự chỉnh lớp bảo vệ và tùy chọn gọi LLM thật bằng key của người dùng.

## Người dùng làm gì?

- Thử các cách diễn đạt, xem kỹ thuật được engine phát hiện, lý do chặn/rò rỉ và bài học đội xanh.
- Ở màn gián tiếp, viết nội dung tài liệu mà một trợ lý giả lập truy xuất; đây là nguồn dữ liệu không đáng tin, không phải system prompt thật.
- Theo dõi kỷ lục ít lượt nhất ở `localStorage`, đổi ngôn ngữ Việt/Anh, bật/tắt âm thanh.

## Chạy và kiểm tra

`npm run serve` (hoặc `python -m http.server 8088`) rồi mở `http://localhost:8088`; ES modules cần HTTP. `npm test` chạy ba test Node: engine, indirect, store. Mặc định không cần cài dependency, API key hay backend.

**Giới hạn:** kết quả trên simulator chỉ phản ánh các regex/luật/điểm của chính app, không dự báo tỷ lệ vượt phòng thủ của mô hình thật. LLM thật chỉ ở sandbox, có mạng và key (hoặc Ollama local); không dùng bí mật thật.
