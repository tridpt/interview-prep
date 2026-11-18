# 07 — Cheat sheet

- **Một câu:** game giáo dục prompt injection, 7 màn direct + 5 màn RAG, simulator tất định.
- **Không backend mặc định:** ES modules trong browser; cần HTTP server cục bộ để tải module.
- **Engine:** `runAttack` cho direct; `runIndirect` cho tài liệu độc; pattern + ngưỡng + phòng thủ mô phỏng.
- **State:** tiến trình/kỷ lục `localStorage`; trạng thái `primed` qua các lượt direct.
- **Sandbox thật:** `llm.js` tùy chọn gọi Gemini/Groq/OpenAI/Ollama, không lưu key trên server app.
- **Chạy:** `npm run serve` → cổng 8088; `npm test` → ba suite Node.
- **Hạn chế:** simulator không chứng minh prompt thực sự phá LLM; không dùng bí mật thật.
