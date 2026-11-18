# 8. Cheat Sheet — Xem nhanh trước giờ G

Một trang tóm tắt. Đọc lướt 5-10 phút trước khi vào phỏng vấn.

---

## 🎯 Pitch 30 giây (học thuộc)
> Agent Talk: nhiều AI có tính cách riêng tự trò chuyện/tranh biện theo thời gian thực, người dùng xem và xen vào. Backend **FastAPI** điều phối gọi **LLM** (chuẩn OpenAI-compatible), **streaming** từng chữ, tóm tắt khi dài, tra cứu web, chấm điểm, bộ nhớ dài hạn. Frontend **JS thuần** lo vòng lặp lượt nói, lưu phiên, xuất file.

## 🧱 Tech stack
- **Backend**: FastAPI, Uvicorn (ASGI), httpx (async), python-dotenv, Pydantic.
- **Frontend**: HTML/CSS/JS thuần; Fetch+Streams, localStorage, SpeechSynthesis, Clipboard.
- **LLM**: Groq (mặc định) / Gemini / OpenRouter — chuẩn OpenAI-compatible.
- **Search**: DuckDuckGo (free) / Tavily (key).
- **Test**: pytest (mock LLM). **Deploy**: Docker (python:3.13-slim).

## 🗂️ 4 lớp backend
`config.py` (cấu hình .env) → `llm.py` (gọi LLM, stream/non-stream) → `search.py` (web, không raise) → `main.py` (điều phối + endpoints).

## 🔌 Endpoint chính
| Endpoint | Việc |
|---|---|
| `turn_stream` | sinh 1 lượt nói (streaming) ⭐ |
| `next_speaker` | LLM chọn người nói tiếp |
| `summarize` | nén ngữ cảnh |
| `score` | chấm điểm 4 tiêu chí + winner |
| `distill_memory` | bộ nhớ dài hạn |
| `session/save` & `session/{id}` | chia sẻ phiên |
| `config` | info provider (không lộ key) |

## 🌊 Luồng streaming (đọc vanh vách)
LLM (`data: delta`) → `chat_stream` (async generator `yield`) → `StreamingResponse` → frontend `body.getReader()` → ghép & hiện dần.

## 🎭 Ánh xạ vai trò (chi tiết tinh tế)
Lời của agent đang nói → `assistant`; lời người khác → `user` "Tên: nội dung"; bối cảnh/luật → `system`. → LLM nhập vai đúng nhân vật.

## 🧠 Quản lý ngữ cảnh
Giữ **8 lượt** cuối nguyên văn; vượt **16 lượt** → tóm tắt dồn các lượt cũ (rolling summary). Gửi: tóm tắt + 8 lượt cuối.

## 🌡️ temperature
~0.9 cho lời thoại (sáng tạo); 0.2-0.3 cho tóm tắt/chấm điểm/chọn người/truy vấn (chính xác).

## 🔎 Tra cứu web (RAG đơn giản)
LLM tự quyết có cần search → search → nhồi ngữ cảnh → footnote nguồn. Lỗi search → trả rỗng, agent vẫn nói (graceful degradation).

## 🏆 Chấm điểm — phòng thủ JSON
Regex bóc `{...}`; parse lỗi → 502; thiếu `overall`/`winner` → tự tính/tự chọn. **Không tin LLM trả đúng định dạng.**

## 🔐 Bảo mật (nhớ 4 điểm)
1. API key chỉ ở backend; `/api/config` trả `configured` chứ không trả key.
2. Chống path traversal: mã phiên `[A-Za-z0-9]{1,32}`.
3. Chống XSS: `escapeHtml` mọi nội dung động; link phải `https?://`.
4. Giới hạn phiên 1MB. **Hạn chế đã biết:** phiên chia sẻ không có auth → chỉ hợp local.

## 🧪 Test
pytest + **mock LLM** (monkeypatch) → offline, miễn phí, deterministic. Test cả nhánh lỗi (history rỗng, JSON sai, mã sai).

## ⚖️ Trade-off chính
- Backend **stateless**, điều phối ở **frontend** → server đơn giản, tương tác tốt; đổi lại payload lớn dần, khó multi-viewer realtime.
- **Vanilla JS** → nhẹ, không build; đổi lại khó mở rộng UI lớn.

## 🚀 Cải thiện tương lai (nếu được hỏi)
Auth + TTL cho phiên chia sẻ; API search ổn định thay scraping; đồng bộ đa thiết bị; rate limit; hàng đợi gọi LLM khi tải cao.

## 💬 Thuật ngữ chốt
- **LLM / token / context window**
- **OpenAI-compatible** (đổi provider = đổi config)
- **streaming / SSE / async generator / ASGI**
- **orchestration / prompt engineering**
- **RAG / hallucination / graceful degradation**
- **stateless / path traversal / XSS / mock**

---

### ✅ Checklist tâm lý trước phỏng vấn
- [ ] Kể được dự án trong 30 giây.
- [ ] Vẽ/nói được luồng 1 lượt nói.
- [ ] Giải thích được streaming end-to-end.
- [ ] Nêu được 2 quyết định kỹ thuật + lý do.
- [ ] Nêu được 2 hạn chế + cách khắc phục.
- [ ] Chuẩn bị 2 câu hỏi ngược lại nhà tuyển dụng.

**Hít thở sâu. Bạn hiểu dự án này. Cứ tự tin kể câu chuyện của mình. 🍀**
