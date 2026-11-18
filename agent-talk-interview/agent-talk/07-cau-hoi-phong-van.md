# 7. Ngân hàng câu hỏi phỏng vấn & trả lời mẫu

60+ câu, chia theo nhóm, từ dễ đến khó. Mỗi câu có **gợi ý trả lời**. Hãy đọc rồi tự diễn đạt lại bằng lời của bạn.

> ⭐ = câu rất hay được hỏi. Ưu tiên luyện kỹ.

---

## A. Tổng quan & động lực (warm-up)

**A1. ⭐ Giới thiệu ngắn gọn về dự án của bạn.**
> Agent Talk là ứng dụng web cho nhiều nhân vật AI có tính cách riêng tự trò chuyện/tranh biện theo thời gian thực, người dùng xem và xen vào điều hướng. Backend FastAPI điều phối việc gọi LLM (qua API chuẩn OpenAI), hỗ trợ streaming từng chữ, tóm tắt khi dài, tra cứu web, chấm điểm và bộ nhớ dài hạn. Frontend là JS thuần, lo vòng lặp lượt nói, lưu phiên và xuất file.

**A2. Vì sao bạn làm dự án này?**
> Để thực hành các bài toán cốt lõi khi xây sản phẩm AI thật: điều phối nhiều lượt gọi LLM, xử lý streaming, quản lý ngữ cảnh dài, và tích hợp dịch vụ ngoài — thay vì chỉ gọi API một phát.

**A3. Đối tượng người dùng là ai?**
> Người tò mò về AI, người muốn brainstorm/tranh biện đa góc nhìn, và bản thân để học. Đây thiên về công cụ khám phá & trình diễn kỹ thuật hơn là sản phẩm thương mại.

**A4. Mất bao lâu để làm? Làm một mình hay nhóm?**
> (Trả lời theo thực tế của bạn — nêu rõ vai trò và phần bạn tự làm.)

---

## B. Kiến trúc & thiết kế

**B1. ⭐ Mô tả kiến trúc tổng thể.**
> Hai phần: frontend tĩnh (HTML/CSS/JS) chạy trên trình duyệt và backend FastAPI. Backend chia 4 lớp: `config` (cấu hình), `llm` (gọi LLM), `search` (tra web), `main` (điều phối + các endpoint). Frontend giữ trạng thái hội thoại và gọi các endpoint REST; phần nói dùng HTTP streaming.

**B2. ⭐ Trạng thái hội thoại nằm ở đâu? Backend có stateful không?**
> Backend gần như **stateless** — mỗi request `turn_stream` tự chứa đủ ngữ cảnh do frontend gửi lên. Trạng thái thật ở frontend (biến `history`, `summary`...) và lưu localStorage. Ngoại lệ duy nhất là các phiên **chia sẻ** được ghi file ở `storage/sessions`.

**B3. Ưu/nhược của thiết kế stateless này?**
> Ưu: server đơn giản, dễ scale ngang, không cần session store. Nhược: payload lớn dần theo hội thoại (đã giảm bằng tóm tắt), và phiên gắn với một trình duyệt (đã giải quyết phần nào bằng chia sẻ link).

**B4. Vì sao chia nhiều endpoint nhỏ thay vì một endpoint to?**
> Tách trách nhiệm, dễ test/debug; frontend chủ động điều phối vòng lặp nên có thể dừng/xen vào bất cứ lúc nào; các tác vụ phụ (tóm tắt, chấm điểm) gọi khi cần.

**B5. Vì sao frontend giữ vòng lặp điều phối chứ không phải backend?**
> Để người dùng kiểm soát trực tiếp: dừng, xen vào, đổi hướng tức thì. Backend chỉ phục vụ từng "mảnh". Nếu để backend chạy trọn vòng lặp thì khó tương tác giữa chừng.

**B6. Nếu phải đưa vòng lặp lên backend thì sao?**
> Sẽ cần kênh hai chiều (WebSocket) để client điều khiển giữa chừng, và một session store ở server. Phức tạp hơn nhưng cho phép nhiều người cùng xem một phiên realtime.

---

## C. LLM & Prompt Engineering

**C1. ⭐ "OpenAI-compatible" nghĩa là gì và vì sao quan trọng với dự án?**
> Là định dạng API chuẩn của OpenAI (`/chat/completions`, body có model/messages/temperature...) mà Groq, Gemini, OpenRouter đều bắt chước. Nhờ vậy chỉ cần một hàm gọi LLM, đổi provider chỉ là đổi base_url/key/model.

**C2. ⭐ Mảng `messages` gồm những vai nào? Dự án ánh xạ thế nào?**
> system (luật/bối cảnh), user (lời người khác), assistant (lời mô hình). Dự án: lời của agent đang nói → assistant; lời người khác → user kèm "Tên: nội dung". Nhờ vậy LLM nhập vai đúng nhân vật.

**C3. temperature là gì? Dự án dùng giá trị nào, vì sao?**
> Độ ngẫu nhiên/sáng tạo. Lời thoại dùng ~0.9 (cần đa dạng); tóm tắt/chấm điểm/chọn người nói/nghĩ truy vấn dùng 0.2-0.3 (cần chính xác, nhất quán).

**C4. Làm sao agent "biết" mình là ai và phải đóng vai gì?**
> Qua system prompt: "Bạn là {tên}. {tính cách}..." cộng luật (ngắn gọn 2-4 câu, tiếng Việt có dấu, không mô tả hành động, không đóng vai người khác), cộng ánh xạ vai trò ở lịch sử.

**C5. "Mục tiêu ngầm" hoạt động ra sao?**
> Thêm vào system prompt một ý đồ riêng kèm dặn "đừng nói lộ liễu, hãy khéo dẫn dắt". LLM cố theo đuổi mục tiêu một cách kín đáo.

**C6. Làm sao ép LLM trả JSON khi chấm điểm? Nếu nó trả sai thì sao?**
> Prompt yêu cầu "chỉ trả JSON dạng {...}, không kèm gì khác". Phòng thủ: regex bóc `{...}`, nếu parse lỗi trả 502; nếu thiếu trường thì tự tính overall/winner. Nguyên tắc: không tin LLM trả đúng định dạng.

**C7. Hallucination là gì? Dự án giảm nó thế nào?**
> Là khi LLM "bịa" thông tin. Dự án giảm bằng tra cứu web (RAG đơn giản) đưa dẫn chứng thật vào ngữ cảnh, kèm dặn "không bịa thêm" và footnote nguồn để kiểm chứng.

---

## D. Streaming & Async

**D1. ⭐ Giải thích luồng streaming end-to-end.**
> LLM gửi từng chunk (dòng `data: {...delta...}`) → `chat_stream` (async generator) yield delta → `StreamingResponse` đẩy về → frontend dùng `response.body.getReader()` đọc và ghép dần vào bong bóng chat.

**D2. async/await là gì? Vì sao cần ở đây?**
> Cho phép nhả quyền điều khiển khi chờ I/O (gọi mạng), để server phục vụ nhiều request cùng lúc mà không block. Quan trọng vì gọi LLM mất nhiều giây.

**D3. async generator là gì?**
> Hàm `async def` có `yield`, sinh dữ liệu dần dần và có thể `await` bên trong. Dùng để stream từng mẩu text từ LLM.

**D4. Vì sao có cả `chat` và `chat_stream`?**
> `chat_stream` cho lời thoại (UX gõ chữ). `chat` (non-stream) cho tác vụ nội bộ cần nguyên kết quả một lần (tóm tắt, chấm điểm, chọn người nói).

**D5. Streaming khác gì WebSocket? Vì sao không dùng WebSocket?**
> Streaming HTTP là một chiều server→client, đơn giản. WebSocket hai chiều, phù hợp realtime liên tục. Ở đây mỗi lượt là một request riêng, HTTP streaming đủ và gọn hơn.

**D6. Người dùng bấm Dừng giữa lúc đang stream thì sao?**
> Frontend đặt `running=false`, vòng đọc luồng phát hiện và gọi `reader.cancel()` để hủy, bong bóng rỗng bị xóa.

---

## E. Quản lý ngữ cảnh

**E1. ⭐ Context window là gì? Hội thoại dài gây vấn đề gì?**
> Là giới hạn token LLM nhìn được mỗi lần. Hội thoại dài có thể vượt giới hạn, chậm và tốn tiền.

**E2. ⭐ Dự án xử lý hội thoại dài thế nào?**
> Rolling summary: giữ nguyên 8 lượt gần nhất; khi vượt 16 lượt thì nén các lượt cũ thành tóm tắt (gộp dồn), rồi chỉ gửi tóm tắt + 8 lượt cuối.

**E3. Vì sao tóm tắt ở frontend mà không phải tự động ở backend mỗi lượt?**
> Vì frontend giữ trạng thái và quyết định khi nào cần. Backend cung cấp endpoint `/api/summarize` như một dịch vụ thuần, được gọi khi frontend thấy đủ dài.

---

## F. Tra cứu web

**F1. ⭐ Tính năng tra cứu hoạt động ra sao?**
> Trước khi nói, hỏi LLM có cần dữ kiện thật không và cho 1 truy vấn; nếu có thì search (Tavily nếu có key, không thì DuckDuckGo), nhồi kết quả vào ngữ cảnh, agent nói kèm footnote nguồn.

**F2. Vì sao để LLM tự quyết có cần search?**
> Tiết kiệm gọi mạng và thời gian; chỉ lấy dữ kiện khi thật cần lập luận có sức nặng.

**F3. Nếu search lỗi/timeout thì sao?**
> `search.py` nuốt mọi lỗi và trả danh sách rỗng → agent vẫn nói bình thường (graceful degradation). Tính năng phụ không được làm sập luồng chính.

**F4. Điểm yếu của cách scraping DuckDuckGo bằng regex?**
> Mong manh — nếu họ đổi HTML thì regex hỏng. Đây là đánh đổi để có search free không cần key. Bền hơn thì dùng Tavily hoặc một API search chính thức.

**F5. Đây có phải RAG không?**
> Đúng, là một dạng RAG đơn giản: lấy dữ liệu ngoài đưa vào ngữ cảnh để trả lời chính xác hơn.

---

## G. Bảo mật

**G1. ⭐ API key được bảo vệ thế nào?**
> Lưu trong `.env` (đã `.gitignore`), chỉ đọc ở backend. Endpoint `/api/config` chỉ trả `configured: true/false`, không bao giờ gửi key thật ra frontend.

**G2. ⭐ Endpoint chia sẻ phiên có rủi ro gì?**
> Không có xác thực — ai có mã đều đọc được phiên. Phù hợp chạy cục bộ. Nếu public cần thêm auth, rate limit và TTL hết hạn phiên. (Nêu được hạn chế này là điểm cộng.)

**G3. Chống path traversal ở đâu?**
> Ở `/api/session/{sid}`: chỉ chấp nhận mã khớp `[A-Za-z0-9]{1,32}`, chặn đường dẫn kiểu `../`.

**G4. Chống XSS thế nào?**
> Frontend `escapeHtml()` mọi nội dung động (tên, lời thoại, tiêu đề nguồn) trước khi nhét vào innerHTML; link nguồn phải khớp `^https?://` mới cho bấm.

**G5. Có giới hạn kích thước phiên không?**
> Có, `MAX_SESSION_BYTES = 1MB`, vượt thì trả 413 — chống lạm dụng lưu trữ.

---

## H. Testing

**H1. ⭐ Bạn test thế nào mà không tốn API key?**
> Dùng pytest và **mock** hàm gọi LLM (`monkeypatch` thay `llm.chat`/`chat_stream` bằng hàm giả). Nhờ vậy test chạy offline, nhanh, miễn phí, ổn định.

**H2. Bạn test những gì?**
> Các endpoint (`/api/config`, summarize, next_speaker, score, distill_memory, turn_stream, session) và các hàm thuần trong `search` (format_results, _clean). Có test các nhánh lỗi: history rỗng, JSON sai, mã phiên không hợp lệ.

**H3. Vì sao mock LLM thay vì gọi thật?**
> LLM trả ngẫu nhiên (khó assert), tốn tiền, cần mạng, có thể rate limit. Mock cho test xác định (deterministic) và tách bạch logic điều phối khỏi dịch vụ ngoài.

**H4. Ví dụ một test thú vị?**
> `test_score_normalizes_overall_and_winner`: mock model trả JSON thiếu `overall` và `winner`, kiểm tra backend tự tính trung bình (vd (8+6+7+9)/4 = 7.5) và chọn người thắng đúng.

---

## I. Triển khai & vận hành

**I1. Chạy local thế nào?**
> `pip install -r requirements.txt`, tạo `.env` với API key, rồi `uvicorn backend.main:app --reload`, mở `127.0.0.1:8000`.

**I2. Docker đóng gói ra sao?**
> Image `python:3.13-slim`, cài requirements trước (tận dụng cache layer), copy backend+frontend, chạy uvicorn ở `0.0.0.0:8000`, truyền API key qua biến môi trường khi `docker run`.

**I3. Vì sao cài dependencies trước rồi mới copy code trong Dockerfile?**
> Tận dụng cache layer Docker: code đổi thường xuyên nhưng dependencies ít đổi, nên đặt bước cài trước giúp build lại nhanh.

---

## J. Câu hỏi mở / tư duy

**J1. ⭐ Thử thách khó nhất khi làm dự án?**
> (Gợi ý) Xử lý streaming end-to-end cho mượt và dừng được giữa chừng; hoặc xử lý LLM trả JSON không chuẩn khi chấm điểm. Kể vấn đề → cách bạn nghĩ ra → kết quả.

**J2. ⭐ Nếu có thêm thời gian bạn sẽ cải thiện gì?**
> Thêm auth + TTL cho phiên chia sẻ; thay scraping DuckDuckGo bằng API search ổn định; đồng bộ phiên/bộ nhớ giữa thiết bị; cho khán giả bình chọn song song với trọng tài AI; thêm rate limit.

**J3. Dự án sẽ "vỡ" ở đâu khi nhiều người dùng cùng lúc?**
> Mỗi request gọi LLM tốn thời gian và có thể bị rate limit từ provider; cần hàng đợi/giới hạn đồng thời và rate limit. Lưu phiên ra file cũng không lý tưởng ở quy mô lớn — nên chuyển sang DB/object storage.

**J4. Nếu provider LLM đổi định dạng API thì sao?**
> Vì đã trừu tượng hóa trong `llm.py`, chỉ cần sửa một chỗ. Nếu định dạng khác hẳn chuẩn OpenAI thì viết một adapter riêng cho provider đó.

**J5. Làm sao đảm bảo agent không lặp lại nhau hoặc lạc đề?**
> Luật prompt (ngắn gọn, đúng vai), cơ chế chọn người nói theo mạch, đạo diễn xen vào, và tóm tắt giữ mạch chính. Có thể cải thiện thêm bằng cách nhắc lại chủ đề định kỳ.

**J6. Trade-off lớn nhất của dự án là gì?**
> Đẩy điều phối về frontend (đơn giản server, tương tác tốt) đổi lại payload lớn và khó nhiều-người-xem-một-phiên realtime. Và chọn vanilla JS (nhẹ, không build) đổi lại khó mở rộng UI phức tạp.

---

## K. Câu hỏi "code cụ thể" (có thể bị soi)

**K1. `build_messages` xử lý trường hợp chưa có lịch sử thế nào?**
> Thêm một message user mồi "Hãy bắt đầu cuộc trò chuyện [về chủ đề...]" để LLM có cái mà phản hồi.

**K2. Vì sao đôi khi thêm "(tiếp tục cuộc trò chuyện)"?**
> Nếu message cuối là vai assistant (lời của chính agent), cần một message user để LLM tiếp lời — nên chèn câu này.

**K3. `get_provider` trả gì nếu id không tồn tại?**
> Fallback về provider `default` (không trả None) → an toàn.

**K4. Header `X-Sources` dùng để làm gì và mã hóa ra sao?**
> Gửi danh sách nguồn web (title+url) về frontend kèm response stream; được `quote` (URL-encode) để an toàn ký tự, frontend `decodeURIComponent` rồi parse JSON.

**K5. Khi chia sẻ link, vì sao loại bỏ `memory`?**
> Bộ nhớ là dữ liệu riêng tư của agent qua các phiên; không nên gửi cho người nhận link.

---

## 🎤 5 câu hay gặp nhất — luyện trước tiên
1. A1 — Giới thiệu dự án.
2. D1 — Luồng streaming end-to-end.
3. C2 — Ánh xạ vai trò messages.
4. E2 — Xử lý hội thoại dài (rolling summary).
5. C6 — Ép & phòng thủ JSON khi chấm điểm.

## ❓ Câu hỏi NÊN HỎI LẠI nhà tuyển dụng (thể hiện sự chủ động)
- Team đang dùng LLM provider nào, có bài toán về chi phí/độ trễ không?
- Cách team xử lý đánh giá chất lượng đầu ra LLM (eval) ra sao?
- Sản phẩm có dùng streaming/RAG không, kiến trúc hiện tại thế nào?
