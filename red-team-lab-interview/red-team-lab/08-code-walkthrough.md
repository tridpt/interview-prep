# 08 — Walkthrough theo code

## Chat trực tiếp

1. `route()` ở `app.js` chọn `level/:id`, lấy cấu hình từ `LEVELS` và dựng UI.
2. Người dùng gửi tin: `runAttack(level, userText, state)` kiểm tra input filter, nhận diện `TECHNIQUES`, tính lực/threshold và judge.
3. Nếu có ý định lộ bí mật, engine áp biến đổi, rồi output filter; trả `response`, `blockedBy` và `state.primed` cho lượt sau.
4. UI lưu tiến trình/ít lượt nhất qua `store` khi hoàn thành.

## Tài liệu gián tiếp

1. Route RAG nạp `RAG_LEVELS`; người dùng nhập nội dung tài liệu (không chat trực tiếp với nạn nhân).
2. `runIndirect()` xét blocklist, pattern, provenance, threshold và output scan; trả câu trả lời giả lập và cờ `achieved`.
3. Màn đội xanh giải thích ranh giới tin cậy; kết quả phụ thuộc luật của engine, không phải phản ứng của một LLM thật.

## Sandbox tùy chọn

`renderSandbox()` đọc cấu hình phòng thủ; mặc định vẫn dùng simulator. Nếu bật model thật, `buildSystemPrompt` + `callLLM` ở `llm.js` gửi request đến provider/Ollama và `detectLeak` kiểm tra phản hồi theo chuỗi. Không lấy đó làm đánh giá bảo mật đầy đủ.
