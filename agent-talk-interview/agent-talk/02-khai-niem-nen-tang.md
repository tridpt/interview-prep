# 2. Các khái niệm nền tảng (lý thuyết)

File này giải thích **mọi thuật ngữ** xuất hiện trong dự án, viết cho người chưa biết gì. Nắm chắc phần này thì bạn sẽ tự tin trước mọi câu hỏi "khái niệm".

---

## 2.1. LLM (Large Language Model) — Mô hình ngôn ngữ lớn

**LLM là gì?** Là một mô hình AI được huấn luyện trên lượng văn bản khổng lồ để **dự đoán từ (token) tiếp theo** trong một chuỗi. Nhờ vậy nó có thể viết văn, trả lời câu hỏi, tranh luận... Ví dụ: GPT (OpenAI), Llama (Meta), Gemini (Google).

**Token là gì?** Là đơn vị văn bản nhỏ mà mô hình xử lý — có thể là một từ, một phần của từ, hay dấu câu. Ví dụ "trò chuyện" có thể bị tách thành nhiều token. LLM tính phí và giới hạn theo số token.

**Trong dự án này:** Mọi câu thoại của agent đều do LLM sinh ra. Code của dự án **không tự nghĩ ra câu nói**, nó chỉ:
1. Xây dựng "lời nhắc" (prompt) mô tả nhân vật + bối cảnh + lịch sử.
2. Gửi prompt cho LLM.
3. Nhận lại câu trả lời và hiển thị.

### Các tham số LLM quan trọng trong dự án

- **`temperature` (độ sáng tạo)**: Số từ 0 đến ~2. Càng cao thì câu trả lời càng ngẫu nhiên/sáng tạo; càng thấp thì càng "an toàn", nhất quán.
  - Dự án dùng `0.9` cho lời thoại (cần sáng tạo, đa dạng).
  - Dùng `0.2-0.3` cho các tác vụ cần chính xác: chọn người nói tiếp, tóm tắt, chấm điểm, nghĩ truy vấn tìm kiếm.
- **`max_tokens`**: Giới hạn độ dài câu trả lời. Dự án dùng `300` cho lời thoại, `40` cho truy vấn tìm kiếm (chỉ cần vài từ khóa), `700` cho chấm điểm (cần JSON dài).
- **`model`**: Tên mô hình cụ thể, ví dụ `llama-3.3-70b-versatile` (Groq).

---

## 2.2. API và "OpenAI-compatible"

**API (Application Programming Interface)** là "cửa giao tiếp" để chương trình này gọi chương trình khác. Ở đây ta gọi API của nhà cung cấp LLM qua HTTP.

**"Chuẩn OpenAI-compatible" nghĩa là gì?** OpenAI định nghĩa một định dạng request/response cho việc chat (endpoint `/chat/completions`, body có `model`, `messages`, `temperature`...). Nhiều nhà cung cấp khác (Groq, Gemini, OpenRouter) **cố tình làm giống y hệt** định dạng đó.

➡️ **Lợi ích cực lớn cho dự án:** Chỉ cần viết **một** hàm gọi LLM, rồi đổi `base_url` + `api_key` + `model` là dùng được với bất kỳ nhà cung cấp nào. Đây là lý do `config.py` cho phép cấu hình nhiều "provider" mà `llm.py` không cần biết đang gọi ai.

**Cấu trúc `messages`** (rất quan trọng): một danh sách các tin nhắn, mỗi tin có `role` và `content`:
- `role: "system"` → chỉ dẫn/bối cảnh (mô tả nhân vật, luật chơi). LLM coi đây là "luật nền".
- `role: "user"` → lời của "người dùng" (trong dự án này là lời của các agent **khác**).
- `role: "assistant"` → lời của chính mô hình (trong dự án là lời của **chính agent đang nói**).

---

## 2.3. Streaming — Truyền theo luồng

**Vấn đề:** LLM sinh câu trả lời mất vài giây. Nếu đợi nó viết xong cả đoạn rồi mới hiện ra, người dùng phải chờ với màn hình trống → trải nghiệm tệ.

**Giải pháp streaming:** LLM gửi về **từng mẩu nhỏ (chunk)** ngay khi sinh ra. Frontend hiển thị dần → ta thấy chữ "chạy ra" như đang gõ. Giống hệt cảm giác dùng ChatGPT.

**Cách hoạt động kỹ thuật (SSE-like):** Khi gửi request có `stream: true`, server LLM trả về nhiều dòng dạng:
```
data: {"choices":[{"delta":{"content":"Xin"}}]}
data: {"choices":[{"delta":{"content":" chào"}}]}
data: [DONE]
```
- Mỗi dòng bắt đầu bằng `data:`.
- Phần `delta.content` là mẩu chữ mới.
- `[DONE]` báo kết thúc.

Đây gần với **SSE (Server-Sent Events)** — một kỹ thuật để server đẩy dữ liệu liên tục về client qua một kết nối HTTP mở.

**Trong dự án:**
- `llm.py` → hàm `chat_stream` đọc từng dòng, bóc lấy `delta.content`, `yield` ra ngoài.
- `main.py` → `StreamingResponse` của FastAPI bọc generator này, đẩy dần về frontend.
- `app.js` → dùng `response.body.getReader()` để đọc luồng, ghép dần vào bong bóng chat.

---

## 2.4. Lập trình bất đồng bộ (async / await)

**Vấn đề:** Gọi API LLM mất nhiều giây. Nếu code chạy "đồng bộ" (synchronous), nó sẽ **đứng chờ** (block) cho tới khi có kết quả, trong lúc đó không làm gì khác được → server không phục vụ được request khác.

**Giải pháp bất đồng bộ:** Dùng `async def` và `await`. Khi gặp `await` (chỗ chờ I/O như gọi mạng), chương trình **tạm nhả quyền điều khiển** để xử lý việc khác, rồi quay lại khi có kết quả. Nhờ vậy một server có thể phục vụ nhiều người cùng lúc mà không cần nhiều luồng (thread).

**Từ khóa cần nhớ:**
- `async def foo()` → định nghĩa một "coroutine" (hàm bất đồng bộ).
- `await something()` → chờ một tác vụ async hoàn tất mà không block toàn bộ chương trình.
- `async for` → lặp qua một luồng dữ liệu async (dùng để đọc streaming).
- `yield` trong `async def` → tạo **async generator** (sinh dữ liệu dần dần, dùng cho streaming).

**Trong dự án:** Mọi endpoint trong `main.py` đều `async def`. Hàm gọi LLM (`chat`, `chat_stream`) và web search đều async. `httpx.AsyncClient` là HTTP client async.

---

## 2.5. FastAPI — Web framework

**FastAPI là gì?** Một framework Python để xây dựng web API, nổi bật vì:
- Hỗ trợ **async** sẵn (chạy trên ASGI).
- **Tự validate dữ liệu** nhờ Pydantic (định nghĩa kiểu dữ liệu request bằng class).
- Tự sinh tài liệu API.
- Hiệu năng cao.

**Các khái niệm FastAPI trong dự án:**
- **Decorator route**: `@app.post("/api/turn_stream")` gắn một hàm với một đường dẫn URL + phương thức HTTP.
- **Pydantic model** (`BaseModel`): ví dụ `class TurnRequest(BaseModel)` — FastAPI tự đọc JSON từ request, kiểm tra kiểu, biến thành object Python. Nếu sai kiểu → tự trả lỗi 422.
- **Response types**: `JSONResponse` (trả JSON), `StreamingResponse` (trả luồng), `FileResponse` (trả file).
- **StaticFiles**: phục vụ file tĩnh (HTML/CSS/JS) của frontend.

**ASGI vs WSGI:** ASGI (Asynchronous Server Gateway Interface) là chuẩn cho server Python **async**; WSGI là chuẩn cũ **đồng bộ**. FastAPI dùng ASGI, chạy bằng **Uvicorn** (một ASGI server).

---

## 2.6. HTTP cơ bản

- **Request/Response**: client gửi yêu cầu, server trả phản hồi.
- **Method**: `GET` (lấy dữ liệu), `POST` (gửi dữ liệu lên).
- **Status code**: `200` OK, `400` lỗi do client (request sai), `404` không tìm thấy, `413` payload quá lớn, `500` lỗi server, `502` lỗi từ dịch vụ phía sau.
- **Header**: thông tin phụ kèm request/response. Dự án dùng header tùy biến `X-Sources` để gửi danh sách nguồn web về frontend.
- **Body**: nội dung chính, ở đây là JSON.

---

## 2.7. Prompt Engineering — Nghệ thuật viết lời nhắc

Vì code không tự nghĩ, **chất lượng dự án phụ thuộc lớn vào prompt**. Các kỹ thuật prompt dùng trong dự án:

1. **System prompt định hình vai diễn**: "Bạn là {tên}. {tính cách}..." + luật trả lời (ngắn gọn 2-4 câu, tiếng Việt có dấu, không mô tả hành động, không đóng vai người khác).
2. **Nhồi ngữ cảnh có cấu trúc**: chủ đề, mục tiêu ngầm, bộ nhớ, tóm tắt, giai đoạn tranh biện — mỗi thứ là một dòng/đoạn riêng trong system prompt.
3. **Ánh xạ vai trò khéo léo**: lời của chính agent → `assistant`; lời người khác → `user` kèm tên "{tên}: {nội dung}". Nhờ vậy LLM "tưởng" mình thật sự là nhân vật đó.
4. **Ép định dạng đầu ra**: khi chấm điểm, prompt yêu cầu "Chỉ trả về JSON hợp lệ dạng {...}, không kèm văn bản khác".
5. **Cho đường thoát**: khi hỏi có cần tìm kiếm không, "Nếu không cần, trả lời chính xác: KHÔNG".

---

## 2.8. Quản lý ngữ cảnh (context window)

**Context window** là giới hạn số token mà LLM "nhìn" được trong một lần gọi. Hội thoại càng dài, lịch sử càng nhiều → có nguy cơ **vượt giới hạn** (và tốn tiền, chậm).

**Giải pháp của dự án:**
- Giữ nguyên văn `KEEP_RECENT = 8` lượt gần nhất.
- Khi tổng số lượt vượt `SUMMARY_WINDOW = 16`, **nén** các lượt cũ thành một bản tóm tắt ngắn (gọi endpoint `/api/summarize`), rồi chỉ gửi: tóm tắt + 8 lượt gần nhất.

Đây là kỹ thuật **rolling summary / sliding window** — rất phổ biến trong các sản phẩm chat LLM thực tế.

---

## 2.9. Một số thuật ngữ khác trong dự án

- **Persona**: bản mô tả tính cách của một agent (vai diễn).
- **Goal (mục tiêu ngầm)**: ý đồ riêng LLM được dặn theo đuổi một cách kín đáo.
- **Orchestration (điều phối)**: việc sắp xếp, điều khiển nhiều lượt gọi LLM theo một logic (ai nói, khi nào tóm tắt, khi nào chấm điểm...). **Đây là phần "ruột" của dự án.**
- **localStorage**: kho lưu trữ key-value ngay trong trình duyệt, dữ liệu còn sau khi đóng tab.
- **TTS (Text-to-Speech)**: chuyển văn bản thành giọng nói; dùng `SpeechSynthesis` của trình duyệt.
- **Path traversal**: lỗ hổng bảo mật khi kẻ xấu dùng đường dẫn kiểu `../../etc/passwd` để đọc file ngoài thư mục cho phép. Dự án chặn bằng cách chỉ cho mã phiên là chữ/số.
- **Mock (trong test)**: thay một hàm thật bằng hàm giả để test không phụ thuộc dịch vụ ngoài.
