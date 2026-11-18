# 1. Tổng quan dự án Agent Talk

## 1.1. Agent Talk là gì?

Agent Talk là một **ứng dụng web** cho phép tạo ra nhiều "agent" (nhân vật AI), mỗi agent có một **tính cách (persona)** riêng, rồi để chúng **tự trò chuyện với nhau** về một chủ đề. Người dùng đóng vai khán giả: ngồi xem cuộc hội thoại diễn ra theo thời gian thực, và có thể "xen vào" với vai trò **đạo diễn** để đổi hướng câu chuyện.

Hãy hình dung như một "talkshow AI": bạn tạo ra 2-3 nhân vật (ví dụ một người lạc quan, một người hoài nghi, một triết gia), cho họ một chủ đề ("AI có thay thế con người không?"), bấm nút, và xem họ tranh luận.

### Điểm cốt lõi cần nhớ
- Không phải chatbot người-với-máy thông thường. Đây là **máy-nói-với-máy**, người chỉ quan sát và điều hướng.
- **Không cần GPU**, không tự host model. Dùng **API LLM miễn phí** (Groq, Gemini, OpenRouter).
- Toàn bộ "trí thông minh" đến từ LLM bên ngoài; phần code của dự án là **lớp điều phối (orchestration)** thông minh xoay quanh LLM đó.

## 1.2. Dự án giải quyết vấn đề / mang lại giá trị gì?

Đây là một dự án **mang tính sáng tạo và trình diễn kỹ thuật** hơn là giải một bài toán kinh doanh cụ thể. Giá trị của nó:

1. **Khám phá hành vi LLM**: Xem các tính cách AI khác nhau phản ứng thế nào trước cùng một chủ đề — hữu ích để hiểu prompt engineering.
2. **Công cụ brainstorm / tranh biện**: Cho 2 agent tranh biện 2 phía của một vấn đề để nhìn ra các góc nhìn mình chưa nghĩ tới.
3. **Giải trí & giáo dục**: Mô phỏng tranh biện có chấm điểm, có bảng xếp hạng.
4. **Trình diễn năng lực kỹ thuật**: streaming, orchestration nhiều lượt, quản lý ngữ cảnh, tích hợp web search — đây mới là phần "ăn điểm" khi phỏng vấn.

> 💡 **Mẹo phỏng vấn**: Khi được hỏi "dự án này để làm gì", đừng chỉ nói "cho vui". Hãy nhấn vào **giá trị kỹ thuật**: "Em xây nó để thực hành điều phối nhiều lượt gọi LLM, xử lý streaming và quản lý ngữ cảnh dài — những bài toán cốt lõi khi làm sản phẩm AI thật."

## 1.3. Các tính năng chính (feature list)

| Tính năng | Mô tả ngắn |
|-----------|------------|
| **Nhiều agent** | 2 agent trở lên cùng tham gia hội thoại nhóm |
| **Streaming** | Xem từng chữ agent "gõ" ra theo thời gian thực |
| **Thứ tự nói** | 3 chế độ: luân phiên / AI tự chọn người nói tiếp / chế độ tranh biện |
| **Tính cách mẫu** | 10 persona dựng sẵn + ô nhập tính cách tự do |
| **Mục tiêu ngầm (goal)** | Mỗi agent có một ý đồ riêng để khéo léo dẫn dắt |
| **Đạo diễn (interject)** | Người dùng xen vào giữa chừng để đổi hướng |
| **Tự tóm tắt** | Khi hội thoại dài, tự nén lại để không vỡ ngữ cảnh |
| **Đọc to (TTS)** | Dùng giọng trình duyệt đọc lời mỗi agent |
| **Bộ nhớ dài hạn** | Agent ghi nhớ điều đã học sau mỗi phiên, dùng lại ở phiên sau |
| **Chấm điểm đa tiêu chí** | Trọng tài AI chấm logic/bằng chứng/sáng tạo/thái độ, chọn người thắng |
| **Bảng xếp hạng tích lũy** | Cộng dồn số lần thắng & điểm TB qua nhiều phiên |
| **Tra cứu web** | Agent tìm dẫn chứng thật (DuckDuckGo free hoặc Tavily) trước khi nói |
| **Sao chép & chia sẻ link** | Tạo link ngắn chia sẻ cả phiên (lưu ở server) |
| **Footnote nguồn** | Câu nào dùng dữ liệu web sẽ kèm link nguồn |
| **Lưu phiên & xuất file** | Tự lưu localStorage, xuất `.txt` / `.md` / `.json` |
| **Nhiều "bộ não"** | Cấu hình 2 provider để mỗi agent dùng một LLM khác nhau |

## 1.4. Công nghệ sử dụng (tech stack)

**Backend (Python):**
- **FastAPI** `0.137.0` — web framework bất đồng bộ, xây dựng REST API.
- **Uvicorn** `0.49.0` — ASGI server để chạy FastAPI.
- **httpx** `0.28.1` — HTTP client bất đồng bộ (async), dùng gọi API LLM và web search.
- **python-dotenv** `1.2.2` — đọc cấu hình (API key) từ file `.env`.
- **Pydantic** — định nghĩa & kiểm tra (validate) dữ liệu request (đi kèm FastAPI).

**Frontend:**
- **HTML + CSS + JavaScript thuần (vanilla)** — không dùng React/Vue. Chạy trực tiếp trên trình duyệt.
- Các Web API trình duyệt: **Fetch API + Streams** (đọc luồng), **localStorage** (lưu phiên), **SpeechSynthesis** (đọc to), **Clipboard API**.

**LLM & dịch vụ ngoài:**
- API chuẩn **OpenAI-compatible**: Groq (mặc định), Google Gemini, OpenRouter.
- Web search: **DuckDuckGo HTML** (free, không key) hoặc **Tavily** (free tier, cần key).

**Test & triển khai:**
- **pytest** `9.1.0` — kiểm thử tự động, LLM được **mock** nên không tốn API key, chạy offline.
- **Docker** — đóng gói chạy bằng image `python:3.13-slim`.

> 💡 **Tại sao chọn vanilla JS không framework?** Câu trả lời tốt: "Dự án tập trung độ phức tạp ở phần điều phối backend và xử lý streaming. Frontend chủ yếu là một vòng lặp trạng thái và DOM, dùng vanilla JS giúp dự án **nhẹ, không build step, dễ deploy** (chỉ cần serve file tĩnh). Nếu sau này UI phức tạp hơn thì có thể chuyển sang framework."

## 1.5. Cách chạy dự án (demo flow)

```bash
# 1. Cài thư viện
pip install -r requirements.txt

# 2. Tạo .env và dán API key (ví dụ Groq)
copy .env.example .env

# 3. Chạy server
uvicorn backend.main:app --reload

# 4. Mở trình duyệt: http://127.0.0.1:8000
```

Luồng sử dụng:
1. Đặt tên + tính cách cho ít nhất 2 agent (có thể chọn persona mẫu).
2. (Tùy chọn) nhập chủ đề và mục tiêu ngầm cho từng agent.
3. Chọn thứ tự nói, số lượt, tốc độ, độ sáng tạo (temperature).
4. Bấm **Bắt đầu** → xem chúng trò chuyện. Dùng ô **Đạo diễn** để xen vào.
5. Bấm **Tiếp tục** để chạy thêm lượt, hoặc lưu/chia sẻ/chấm điểm.

## 1.6. Cấu trúc thư mục

```
agent-talk/
├── backend/
│   ├── main.py      # FastAPI: điều phối lượt nói, tóm tắt, chọn người nói, chấm điểm
│   ├── llm.py       # Gọi LLM (chuẩn OpenAI-compatible, có streaming)
│   ├── search.py    # Tra cứu web (Tavily nếu có key, không thì DuckDuckGo)
│   └── config.py    # Đọc cấu hình provider từ .env
├── frontend/
│   ├── index.html   # Giao diện cấu hình + khung chat
│   ├── app.js       # Vòng lặp hội thoại, lưu phiên, xuất file
│   ├── personas.js  # Thư viện tính cách mẫu
│   └── style.css
├── tests/           # Test pytest (LLM được mock)
├── storage/sessions # Nơi lưu các phiên được chia sẻ (file JSON)
├── Dockerfile
├── .env.example
├── requirements.txt
└── requirements-dev.txt
```

**Triết lý thiết kế:** Tách bạch rõ ràng — `config` (cấu hình) ← `llm`/`search` (giao tiếp dịch vụ ngoài) ← `main` (điều phối logic) ← `frontend` (giao diện). Mỗi file một trách nhiệm (separation of concerns).
