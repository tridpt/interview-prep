# 03 — Kiến trúc hệ thống

```text
index.html → app.js (hash router, màn, UI, tiến trình)
  ├─ levels.js + engine.js: 7 màn chat trực tiếp, runAttack/checkGuess
  ├─ indirect.js: 5 màn tài liệu/RAG, runIndirect
  ├─ store.js: màn đã qua + kỷ lục trong localStorage
  ├─ i18n.js / audio.js: VI/EN, âm thanh
  └─ llm.js: chỉ sandbox tùy chọn → Gemini/Groq/OpenAI hoặc Ollama local
```

`app.js` lấy route từ hash, chọn màn direct/RAG/sandbox/bảng xếp hạng, render UI và gọi engine tương ứng. `engine.js` và `indirect.js` là mô phỏng chạy tại client, không gửi prompt đến backend. `llm.js` gọi provider trực tiếp từ trình duyệt khi người dùng chọn model thật; key ở bộ nhớ của trang, có rủi ro khi dùng trên máy/trình duyệt không tin cậy.

**Phân biệt:** 7 màn direct nhập tin nhắn vào bot giả; 5 màn RAG nhập **tài liệu** độc vào luồng truy xuất giả. Cả hai đều phục vụ giáo dục; đừng diễn giải điểm số engine là khả năng jailbreak LLM thật.
