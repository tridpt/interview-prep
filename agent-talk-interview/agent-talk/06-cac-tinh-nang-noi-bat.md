# 6. Phân tích sâu các tính năng "ăn điểm"

File này giúp bạn **kể chuyện ấn tượng** về các tính năng phức tạp. Mỗi tính năng đi kèm: ý tưởng, cách làm, và "vì sao nó khó/hay".

---

## 6.1. Streaming từng chữ ⭐ (tính năng quan trọng nhất)

**Ý tưởng:** Người dùng thấy agent "gõ chữ" như thật, không phải chờ màn hình trống.

**Chuỗi kỹ thuật xuyên suốt:**
```
LLM (stream=true) → llm.chat_stream (async generator, yield delta)
  → StreamingResponse (FastAPI) → HTTP luồng text
  → fetch + response.body.getReader() (frontend) → ghép & hiện dần
```

**Vì sao hay:** Đây là một luồng **end-to-end streaming** thật sự, đụng tới async generator, ASGI streaming response, và Streams API của trình duyệt. Biết giải thích trọn chuỗi này chứng tỏ bạn hiểu cả backend lẫn frontend.

**Câu có thể bị hỏi vặn:**
- *"Sao không dùng WebSocket?"* → Streaming một chiều (server→client) thì HTTP streaming/SSE đơn giản và đủ dùng; WebSocket phù hợp khi cần hai chiều realtime liên tục. Ở đây mỗi lượt là một request riêng, HTTP streaming gọn hơn.
- *"Dừng giữa chừng thế nào?"* → Frontend kiểm tra cờ `running`, gọi `reader.cancel()` để hủy đọc luồng.

---

## 6.2. Điều phối nhiều agent & chọn người nói tiếp ⭐

**Ba chế độ thứ tự nói:**
1. **Luân phiên**: đơn giản, lấy người kế tiếp trong danh sách.
2. **AI tự chọn** (`/api/next_speaker`): gửi diễn biến cho LLM, hỏi "ai nên nói tiếp" → tạo cảm giác hội thoại tự nhiên. Có **fallback luân phiên** nếu LLM trả tên lạ.
3. **Tranh biện**: 3 giai đoạn cố định (mở màn → phản biện → kết luận), mỗi giai đoạn đổi prompt.

**Vì sao hay:** Thể hiện tư duy **orchestration** — biến nhiều lời gọi LLM rời rạc thành một cuộc hội thoại có mạch. Phần fallback thể hiện tư duy phòng thủ (LLM có thể trả sai).

---

## 6.3. Ánh xạ vai trò để LLM "nhập vai" ⭐

**Vấn đề tinh tế:** API LLM chỉ có 3 vai: system/user/assistant. Nhưng ta có nhiều agent. Làm sao để agent A "tưởng" mình là A?

**Giải pháp:** Khi dựng prompt cho agent A:
- Lời của **A** trong lịch sử → gán vai `assistant` (LLM hiểu "đây là lời tôi đã nói").
- Lời của **mọi người khác** → gán vai `user`, kèm tên `"Tên: nội dung"`.

➡️ LLM được "đánh lừa" một cách hợp lý rằng nó chính là A đang nói chuyện với những người khác. **Đây là chi tiết tinh tế nhất của prompt design trong dự án.**

---

## 6.4. Quản lý ngữ cảnh dài bằng rolling summary ⭐

**Vấn đề:** Hội thoại dài → vượt context window, chậm, tốn tiền.

**Giải pháp:** Giữ 8 lượt gần nhất nguyên văn; khi vượt 16 lượt → nén các lượt cũ thành tóm tắt (gộp dồn với tóm tắt trước). Mỗi lượt mới chỉ gửi: `tóm tắt + 8 lượt cuối`.

**Vì sao hay:** Đây là kỹ thuật thật sự dùng trong sản phẩm LLM thương mại. Biết khái niệm context window + cách nén là điểm cộng lớn.

---

## 6.5. Tra cứu web có điều kiện (LLM tự quyết) ⭐

**Luồng 2 bước thông minh:**
1. Trước khi nói, hỏi LLM: "Có cần dữ kiện thật không? Nếu có, cho 1 truy vấn ngắn; nếu không, trả lời KHÔNG."
2. Nếu có → search → nhồi kết quả vào ngữ cảnh → agent nói có dẫn chứng + footnote nguồn.

**Vì sao hay:**
- Không phải lúc nào cũng search (tiết kiệm, nhanh) — **để LLM tự quyết**.
- Đây là một dạng **RAG (Retrieval-Augmented Generation)** đơn giản: lấy dữ liệu ngoài bổ trợ cho câu trả lời.
- `search.py` **không bao giờ raise** → tính năng phụ không làm sập luồng chính (graceful degradation).

**Có thể bị hỏi:** *"RAG là gì?"* → Kỹ thuật cho LLM "tra cứu" nguồn dữ liệu ngoài rồi đưa vào ngữ cảnh để trả lời chính xác hơn, giảm "bịa" (hallucination).

---

## 6.6. Trọng tài AI chấm điểm + xử lý JLM trả JSON không chuẩn ⭐

**Ý tưởng:** Một lời gọi LLM riêng đóng vai trọng tài trung lập, chấm 4 tiêu chí (logic/bằng chứng/sáng tạo/thái độ), chọn người thắng.

**Phần kỹ thuật đáng nói — phòng thủ khi parse:**
- Dùng regex bóc khối `{...}` phòng khi model bọc bằng ```` ```json ````.
- Nếu parse lỗi → trả 502 + raw để debug.
- Nếu model **quên** `overall` → tự tính trung bình; quên `winner` → tự chọn người điểm cao nhất.

**Vì sao hay:** Thể hiện nguyên tắc vàng khi làm với LLM: **"Không bao giờ tin LLM trả đúng định dạng"** — luôn có lớp chuẩn hóa/phòng thủ. Đây là điểm rất "thực chiến".

---

## 6.7. Bộ nhớ dài hạn (long-term memory)

**Ý tưởng:** Sau mỗi phiên, mỗi agent "chắt lọc" những gì đã học thành vài gạch đầu dòng, lưu lại, và đưa vào prompt ở phiên sau → agent có vẻ "nhớ" và "trưởng thành".

**Cách làm:** `/api/distill_memory` gửi tính cách + trí nhớ cũ + hội thoại → LLM viết lại trí nhớ (gộp cũ+mới, bỏ vụn vặt, tối đa 5 dòng).

**Quyền riêng tư:** Khi chia sẻ link, bộ nhớ riêng bị loại bỏ (không gửi đi).

---

## 6.8. Mục tiêu ngầm (hidden goal)

Mỗi agent có thể được giao một ý đồ kín đáo ("dẫn câu chuyện về phía X"), dặn LLM "đừng nói lộ liễu, hãy khéo léo dẫn dắt". Tạo chiều sâu kịch tính cho hội thoại — và là một thử nghiệm thú vị về khả năng "đóng vai có mục tiêu" của LLM.

---

## 6.9. Đa provider — mỗi agent một "bộ não"

Nhờ chuẩn OpenAI-compatible, có thể cấu hình 2 provider (vd Groq + Gemini) và cho mỗi agent dùng model khác nhau → so sánh "tính cách" của các model. Backend chỉ cần đổi `base_url/key/model`.

---

## 6.10. Bảng tổng hợp "tính năng ↔ khái niệm kỹ thuật" (học thuộc)

| Tính năng | Khái niệm kỹ thuật bạn thể hiện |
|-----------|-------------------------------|
| Streaming từng chữ | async generator, StreamingResponse, Streams API, SSE |
| Chọn người nói tiếp | orchestration, fallback phòng thủ |
| Ánh xạ vai trò | prompt engineering, mô hình messages của LLM |
| Rolling summary | quản lý context window, sliding window |
| Tra cứu web có điều kiện | RAG, graceful degradation, scraping |
| Chấm điểm JSON | xử lý đầu ra LLM không tin cậy, validate |
| Bộ nhớ dài hạn | distillation, quyền riêng tư dữ liệu |
| Đa provider | trừu tượng hóa qua chuẩn chung, cấu hình tách biệt |
| Chia sẻ link | stateless server, chống path traversal |
| escapeHtml | bảo mật XSS phía client |

> 💡 **Chiến lược kể chuyện:** Nếu được hỏi "tính năng nào em tự hào nhất?", chọn **streaming end-to-end** hoặc **xử lý JSON chấm điểm phòng thủ** — cả hai đều cho thấy chiều sâu kỹ thuật và tư duy thực chiến.
