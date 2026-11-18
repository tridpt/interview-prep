# 4. Backend chi tiết (giải thích từng file)

File này "mổ xẻ" từng phần backend. Đọc kèm code thật trong `agent-talk/backend/`.

---

## 4.1. `config.py` — Cấu hình provider

**Nhiệm vụ:** Đọc API key & thông tin model từ file `.env`, định nghĩa các "provider" (nhà cung cấp LLM).

**Điểm chính:**
```python
ROOT = Path(__file__).resolve().parent.parent
load_dotenv(ROOT / ".env")   # nạp biến môi trường từ .env
```
- Dùng `pathlib.Path` để tính đường dẫn gốc dự án một cách chắc chắn (không phụ thuộc thư mục đang chạy).
- `load_dotenv` đọc file `.env` thành biến môi trường.

```python
PROVIDERS = {
    "default": _provider("LLM_", "https://api.groq.com/openai/v1", "llama-3.3-70b-versatile", "Groq"),
    "alt":     _provider("LLM2_", "", "", "Provider phụ"),
}
```
- Hai provider: `default` (Groq) và `alt` (tùy chọn, ví dụ Gemini).
- Hàm `_provider(prefix, ...)` đọc các biến `{prefix}BASE_URL`, `{prefix}API_KEY`, ... có giá trị mặc định nếu thiếu.

**Hai hàm quan trọng:**
- `get_provider(id)`: lấy provider theo id, **fallback về default** nếu không có → an toàn, không bao giờ trả None.
- `public_providers()`: trả thông tin **an toàn** cho frontend — có `configured: bool(api_key)` nhưng **không gửi key thật ra ngoài**. (Điểm bảo mật đáng nói khi phỏng vấn.)

> 💡 **Tại sao tách config riêng?** Để chỗ duy nhất quản lý cấu hình nhạy cảm; các module khác chỉ "xin" provider chứ không tự đọc env. Dễ thay đổi, dễ test.

---

## 4.2. `llm.py` — Lớp gọi LLM

**Nhiệm vụ:** Đóng gói việc gọi API LLM theo chuẩn OpenAI-compatible, cả streaming và không streaming, xử lý lỗi gọn gàng.

**`LLMError`**: một exception riêng. Mọi lỗi (mạng, status code, sai định dạng) đều được "gói" lại thành `LLMError` → tầng trên (`main.py`) chỉ cần bắt một loại lỗi này.

**`_prepare(...)`**: hàm dùng chung, dựng `url`, `headers` (gắn `Authorization: Bearer {key}`), và `payload` (model, messages, temperature, max_tokens, stream). Nếu provider chưa có key → raise `LLMError` ngay.

**`chat(...)` (non-streaming):**
```python
async with httpx.AsyncClient(timeout=60) as client:
    resp = await client.post(url, headers=headers, json=payload)
...
return data["choices"][0]["message"]["content"].strip()
```
- Dùng cho các tác vụ "một phát ăn ngay": chọn người nói, tóm tắt, chấm điểm, bộ nhớ, nghĩ truy vấn.
- Bắt `httpx.HTTPError` (lỗi mạng) và status code != 200, biến thành `LLMError`.

**`chat_stream(...)` (streaming — async generator):**
```python
async with client.stream("POST", url, ...) as resp:
    async for line in resp.aiter_lines():
        if not line.startswith("data:"): continue
        data = line[len("data:"):].strip()
        if data == "[DONE]": break
        obj = json.loads(data)
        delta = obj["choices"][0]["delta"].get("content")
        if delta: yield delta
```
- Đọc từng dòng SSE, bóc `delta.content`, `yield` ra dần.
- `timeout=120` (cao hơn vì stream lâu hơn).
- Bọc trong `try/except` để mọi lỗi parse từng dòng được **bỏ qua an toàn** (`continue`) — một dòng hỏng không làm chết cả luồng.

> 💡 **Tại sao có cả 2 hàm chat và chat_stream?** Vì 2 nhu cầu khác nhau: lời thoại cần streaming (UX gõ chữ); các tác vụ nội bộ (tóm tắt, chấm điểm) cần nguyên kết quả một lần để xử lý tiếp → non-streaming gọn hơn.

---

## 4.3. `search.py` — Tra cứu web

**Nhiệm vụ:** Cho agent tìm dẫn chứng thật. Ưu tiên Tavily (nếu có key), fallback DuckDuckGo (free, không key).

**Triết lý cốt lõi (nói khi phỏng vấn):** *"Tra cứu là tính năng phụ trợ, không được làm sập luồng chính."* Vì vậy:
```python
async def web_search(query, max_results=4):
    ...
    try:
        if TAVILY_KEY:
            res = await _tavily(...)
            if res: return res
        return await _duckduckgo(...)
    except (httpx.HTTPError, Exception):
        return []   # mọi lỗi → trả rỗng, agent vẫn nói bình thường
```
**Không bao giờ raise ra ngoài** — đây là quyết định thiết kế có chủ đích (graceful degradation).

**`_duckduckgo(...)`**: gọi `https://html.duckduckgo.com/html/`, rồi **dùng regex bóc** tiêu đề/link/snippet từ HTML. (Đây là "scraping" trang HTML, vì DuckDuckGo bản free không có API JSON.)
- `_clean()`: xóa thẻ HTML (`_TAG_RE`) và giải mã ký tự HTML entity (`html.unescape`).

**`_tavily(...)`**: gọi API JSON chuẩn, sạch hơn.

**`format_results(...)`**: định dạng kết quả thành text `[1] Tiêu đề — snippet (nguồn: url)` để nhồi vào prompt LLM.

> ⚠️ **Điểm yếu thành thật được**: scraping HTML DuckDuckGo bằng regex là **mong manh** — nếu họ đổi cấu trúc HTML thì regex hỏng. Đây là đánh đổi để có tính năng free không cần key. Giải pháp bền hơn: dùng Tavily hoặc một API search chính thức.

---

## 4.4. `main.py` — Lớp điều phối (trái tim dự án)

### Các Pydantic model (định nghĩa & validate request)
- `Agent`: name, persona, provider, goal, memory.
- `TurnRequest`, `NextRequest`, `SummarizeRequest`, `ScoreRequest`, `MemoryRequest`: mỗi endpoint một schema rõ ràng, có giá trị mặc định.

### `build_messages(req)` — Hàm dựng prompt (quan trọng nhất)
Trả về `(me, messages)`:
1. Tìm agent đang nói (`_find`), liệt kê người khác.
2. Dòng "ai đang trò chuyện": nhóm (>2 agent) hay 1-1.
3. Ghép các thành phần system prompt: tính cách + chủ đề + **mục tiêu ngầm** (dặn đừng nói lộ) + **bộ nhớ** + **giai đoạn tranh biện** (mở màn/phản biện/kết luận) + luật trả lời (ngắn gọn 2-4 câu, tiếng Việt có dấu, không mô tả hành động, không đóng vai người khác).
4. Nếu có `summary` → thêm một message system "Tóm tắt diễn biến trước đó".
5. **Ánh xạ vai trò**: duyệt `history`, lời của `me` → `assistant`, lời người khác → `user` dạng `"Tên: nội dung"`.
6. Xử lý mép: nếu chưa có lịch sử → thêm câu mồi "Hãy bắt đầu..."; nếu lượt cuối là assistant → thêm "(tiếp tục cuộc trò chuyện)" (LLM cần kết thúc bằng user để trả lời).

### `maybe_search_context(req, me)` — Tra cứu có điều kiện
- Chỉ chạy khi `allow_search=True`.
- **Hai bước**: (1) hỏi LLM "có cần tra cứu không, nếu có cho 1 truy vấn" (temperature thấp 0.2, max_tokens 40); nếu LLM trả "KHÔNG" → bỏ qua. (2) nếu có truy vấn → gọi `search.web_search`, định dạng, trả về một message system.
- **Không bao giờ raise** (bọc try/except). Đây là kỹ thuật **"để LLM tự quyết có cần dữ kiện thật không"** — thông minh và tiết kiệm.

### `/api/turn_stream` — Endpoint trung tâm
- Dựng messages → kiểm tra api_key → (tùy chọn) chèn ngữ cảnh tra cứu vào vị trí 1 → `StreamingResponse` bọc generator gọi `chat_stream`.
- Lỗi LLM trong lúc stream → `yield "\n[LỖI] ..."` (không làm sập kết nối).
- Nếu có nguồn → gắn header `X-Sources` (JSON đã `quote` để an toàn ký tự).

### `/api/next_speaker` — Chọn người nói tiếp
- Gửi diễn biến gần đây + người vừa nói, hỏi LLM "ai nên nói tiếp" (temperature 0.3).
- **Khớp tên** trong câu trả lời; nếu không khớp → **fallback luân phiên** (lấy người kế tiếp trong danh sách). Luôn đảm bảo trả về một tên hợp lệ.

### `/api/summarize` — Nén ngữ cảnh
- Tóm tắt 3-5 câu, có gộp với tóm tắt trước đó (rolling summary). temperature 0.3.

### `/api/score` — Trọng tài AI (đáng chú ý về xử lý JSON)
- Prompt yêu cầu trả **JSON** với 4 tiêu chí (logic, evidence, creativity, attitude) + overall + winner.
- **Phòng thủ khi parse**:
  - `re.search(r"\{.*\}", text, re.DOTALL)` để bóc JSON nếu model lỡ bọc bằng ```` ```json ````.
  - Nếu `json.loads` lỗi → trả 502 kèm `raw` (500 ký tự đầu) để debug.
  - **Chuẩn hóa**: nếu model quên `overall` → tự tính trung bình 4 tiêu chí; nếu quên `winner` → tự chọn người overall cao nhất.
- ➡️ Đây là ví dụ tuyệt vời về **"đừng tin LLM trả đúng định dạng"** — luôn có lớp phòng thủ.

### `/api/distill_memory` — Bộ nhớ dài hạn
- Sau mỗi phiên, với mỗi agent: gửi tính cách + trí nhớ cũ + hội thoại, yêu cầu viết lại trí nhớ (tối đa 5 gạch đầu dòng), gộp cũ + mới, bỏ vụn vặt.
- Nếu hội thoại rỗng → trả lại trí nhớ cũ (không gọi LLM vô ích).

### `/api/session/save` và `/api/session/{sid}` — Chia sẻ phiên
- **save**: kiểm tra kích thước (`MAX_SESSION_BYTES = 1MB`, chống lạm dụng) → tạo mã ngẫu nhiên `uuid4().hex[:10]` → ghi file JSON.
- **get**: **chống path traversal** bằng `re.fullmatch(r"[A-Za-z0-9]{1,32}", sid)` — chỉ chấp nhận chữ/số. Không tồn tại → 404; lỗi đọc → 500.

### Phục vụ frontend
- `@app.get("/")` trả `index.html`.
- `app.mount("/", StaticFiles(...))` phục vụ toàn bộ file tĩnh.

> 💡 **Câu hỏi cài bẫy hay gặp:** "Endpoint chia sẻ phiên có bảo mật không?" → Trả lời thật: README ghi rõ **không có xác thực**, ai có mã đều đọc được, phù hợp chạy cục bộ. Nếu public ra internet cần thêm auth + rate limit + TTL (hết hạn phiên). Việc bạn **biết và nêu được hạn chế** sẽ ghi điểm hơn là giấu.

---

## 4.5. Tổng kết các quyết định kỹ thuật trong backend

| Quyết định | Lý do |
|-----------|-------|
| Một lớp `llm.py` chung cho mọi provider | Nhờ chuẩn OpenAI-compatible → đổi provider chỉ là đổi config |
| Bọc mọi lỗi LLM thành `LLMError` | Tầng trên xử lý lỗi nhất quán, đơn giản |
| `search.py` không bao giờ raise | Tính năng phụ không được làm sập luồng chính (graceful degradation) |
| LLM tự quyết có cần search không | Tiết kiệm gọi mạng, dữ kiện chỉ lấy khi thật cần |
| Lớp phòng thủ khi parse JSON chấm điểm | LLM không đảm bảo đúng định dạng |
| Backend stateless (trừ phiên chia sẻ) | Đơn giản, dễ scale; state thật ở frontend |
| Chặn path traversal mã phiên | Bảo mật cơ bản cho thao tác đọc file |
