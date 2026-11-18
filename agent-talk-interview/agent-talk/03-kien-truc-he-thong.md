# 3. Kiến trúc hệ thống

## 3.1. Sơ đồ tổng thể

```
┌─────────────────────────────────────────────────────────────┐
│                        TRÌNH DUYỆT                            │
│  ┌─────────────┐  ┌──────────┐  ┌──────────────────────────┐ │
│  │ index.html  │  │ style.css│  │ app.js (bộ não frontend) │ │
│  │ (giao diện) │  │ (giao    │  │ - vòng lặp lượt nói       │ │
│  │             │  │  diện)   │  │ - lưu phiên (localStorage)│ │
│  │             │  │          │  │ - đọc streaming, TTS      │ │
│  └─────────────┘  └──────────┘  └──────────────────────────┘ │
│         personas.js (10 tính cách mẫu)                        │
└───────────────────────────┬─────────────────────────────────┘
                            │  HTTP (fetch) — JSON & luồng text
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  BACKEND — FastAPI (main.py)                  │
│   Lớp ĐIỀU PHỐI: các endpoint /api/*                          │
│   turn_stream · next_speaker · summarize · score ·           │
│   distill_memory · session/save · session/{id} · config      │
│        │                          │                           │
│        ▼                          ▼                           │
│   ┌─────────┐               ┌───────────┐                     │
│   │ llm.py  │               │ search.py │                     │
│   │ gọi LLM │               │ web search│                     │
│   └────┬────┘               └─────┬─────┘                     │
│        │   đọc cấu hình từ config.py (.env)                   │
└────────┼─────────────────────────┼───────────────────────────┘
         │                         │
         ▼                         ▼
  ┌──────────────┐         ┌─────────────────────┐
  │ LLM Provider │         │ DuckDuckGo / Tavily  │
  │ Groq/Gemini  │         │ (tra cứu web)        │
  └──────────────┘         └─────────────────────┘
```

## 3.2. Phân lớp & trách nhiệm (separation of concerns)

| Lớp | File | Trách nhiệm | Không làm gì |
|-----|------|-------------|--------------|
| **Cấu hình** | `config.py` | Đọc `.env`, định nghĩa các provider | Không gọi mạng |
| **Giao tiếp LLM** | `llm.py` | Gọi API LLM (stream & non-stream), bắt lỗi | Không biết logic hội thoại |
| **Giao tiếp web** | `search.py` | Tra cứu web, làm sạch HTML | Không bao giờ raise lỗi ra ngoài |
| **Điều phối** | `main.py` | Xây prompt, sắp lượt nói, tóm tắt, chấm điểm, lưu phiên | Không gọi mạng trực tiếp (qua llm/search) |
| **Giao diện** | `frontend/*` | Vòng lặp UI, hiển thị, lưu phiên client | Không giữ API key |

**Điểm cốt lõi để nói khi phỏng vấn:** Backend là **lớp điều phối "mỏng nhưng thông minh"**. Nó không chứa AI — nó dàn xếp khi nào gọi AI, gọi với ngữ cảnh gì, và xử lý kết quả. Trí tuệ nằm ở LLM; **giá trị kỹ thuật của dự án nằm ở cách điều phối**.

## 3.3. Luồng dữ liệu một lượt nói (quan trọng nhất — học kỹ)

Đây là luồng bạn nên kể vanh vách. Khi một agent nói một lượt:

```
1. [Frontend] app.js gom dữ liệu: danh sách agent, người nói, chủ đề,
   lịch sử (từ điểm đã tóm tắt trở đi), summary, temperature, allow_search.
        │ POST /api/turn_stream
        ▼
2. [Backend main.py] build_messages(): dựng mảng `messages`:
   - system prompt: "Bạn là {tên}. {tính cách}..." + chủ đề + mục tiêu
     ngầm + bộ nhớ + giai đoạn tranh biện + luật trả lời.
   - (nếu có) system: "Tóm tắt diễn biến trước đó: ..."
   - lịch sử: lời mình → role assistant; lời người khác → role user "Tên: ..."
        │
        ▼
3. [Backend] maybe_search_context(): NẾU allow_search:
   - hỏi LLM "có cần tra cứu không? nếu có, cho 1 truy vấn ngắn".
   - nếu có → search.web_search() → chèn kết quả vào messages (vị trí 1).
        │
        ▼
4. [Backend] kiểm tra provider có API key chưa (nếu chưa → trả 400).
        │
        ▼
5. [Backend] llm.chat_stream(): gọi API LLM với stream=true,
   đọc từng dòng "data:", bóc delta.content, yield ra.
   FastAPI bọc trong StreamingResponse, kèm header X-Sources (nếu có nguồn).
        │ luồng text chạy về
        ▼
6. [Frontend] đọc luồng bằng reader.read(), ghép dần vào bong bóng chat
   (thấy chữ chạy ra). Đọc header X-Sources để hiện footnote nguồn.
        │
        ▼
7. [Frontend] lưu lượt vào `history`, lưu localStorage, đọc to (TTS nếu bật),
   rồi gọi pickNext() để chọn người nói tiếp → lặp lại.
```

## 3.4. Các endpoint API (bảng tra cứu nhanh)

| Endpoint | Method | Chức năng |
|----------|--------|-----------|
| `/api/turn_stream` | POST | Sinh một lượt nói (streaming) — endpoint trung tâm |
| `/api/next_speaker` | POST | Hỏi LLM nên cho ai nói tiếp (chế độ "Tự chọn") |
| `/api/summarize` | POST | Tóm tắt đoạn hội thoại cũ để nén ngữ cảnh |
| `/api/score` | POST | Trọng tài AI chấm điểm 4 tiêu chí, chọn người thắng |
| `/api/distill_memory` | POST | Chắt lọc "bộ nhớ dài hạn" cho một agent sau phiên |
| `/api/session/save` | POST | Lưu phiên chia sẻ, trả mã ngắn |
| `/api/session/{sid}` | GET | Đọc lại phiên đã chia sẻ theo mã |
| `/api/config` | GET | Trả thông tin provider (an toàn, không lộ key) |
| `/` và `/...` | GET | Phục vụ frontend (index.html + file tĩnh) |

## 3.5. Vì sao chia thành nhiều endpoint nhỏ thay vì một endpoint to?

Câu hỏi phỏng vấn hay gặp. Trả lời:
- **Tách trách nhiệm**: mỗi endpoint một việc → dễ test, dễ debug.
- **Frontend chủ động điều phối**: vòng lặp hội thoại nằm ở client, server chỉ "phục vụ từng mảnh". Nhờ vậy người dùng có thể dừng, xen vào, đổi hướng bất cứ lúc nào.
- **Linh hoạt**: tóm tắt/chấm điểm/bộ nhớ là các tác vụ độc lập, gọi khi cần, không bắt buộc.

## 3.6. Trạng thái (state) nằm ở đâu?

Quan trọng: **Backend gần như không giữ trạng thái (stateless)**, trừ các file phiên đã chia sẻ trong `storage/sessions`.
- Mỗi request `turn_stream` đều **tự chứa đủ ngữ cảnh** (agent, lịch sử, tóm tắt) do frontend gửi lên.
- **Trạng thái thật nằm ở frontend**: biến `history`, `summary`, `lastSpeaker`... và được lưu vào `localStorage`.

➡️ **Ưu điểm**: server đơn giản, dễ mở rộng theo chiều ngang (scale), không cần session store phức tạp.
➡️ **Nhược điểm**: payload mỗi request lớn dần theo hội thoại (đã giảm nhẹ bằng tóm tắt); phiên gắn với một trình duyệt (đã giải quyết phần nào bằng tính năng chia sẻ link).
