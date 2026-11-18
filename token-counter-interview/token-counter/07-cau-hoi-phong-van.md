# 07 — Câu hỏi phỏng vấn (kèm trả lời mẫu)

---

## A. Tổng quan (1–10)

**1. Dự án làm gì trong một câu?**
Web app local đọc log các công cụ AI để tổng hợp usage thật khi có metadata, và ước tính token cho văn bản bằng tiktoken.

**2. Hai chế độ chính?**
(1) Quét log local lấy usage thật; (2) ước tính token thủ công bằng tiktoken cho văn bản dán vào.

**3. Ai dùng?**
Người dùng các công cụ lập trình AI muốn theo dõi mình đã tiêu bao nhiêu token/credit.

**4. Công nghệ chính?**
FastAPI + uvicorn, tiktoken, Pydantic, thư viện chuẩn Python cho quét log; frontend JS thuần.

**5. Vì sao đếm token quan trọng?**
Vì nhà cung cấp tính tiền theo token; đếm trước giúp dự trù chi phí.

**6. Các nguồn log hỗ trợ?**
Claude Code (token nếu có JSONL/telemetry), Kiro (credit + % context), Google Antigravity (token nếu log có metadata), và dán JSON API thủ công.

**7. Các endpoint chính?**
`/api/health`, `/api/models`, `/api/count` (POST), `/api/usage/scan`.

**8. Dữ liệu có rời máy không?**
Không. Quét chỉ-đọc, hoàn toàn local, không gửi mạng.

**9. Điểm trung thực của app?**
Không bịa token khi nhà cung cấp chỉ lộ credit; nói rõ tiktoken là ước tính cho model không phải OpenAI.

**10. Hai mảng có phụ thuộc nhau không?**
Không — thiếu tiktoken thì phần quét vẫn chạy, và ngược lại.

---

## B. Khái niệm nền tảng (11–26)

**11. Token là gì?**
Mẩu văn bản — đơn vị model đọc và tính tiền (có thể là từ, phần từ, dấu câu).

**12. Tokenizer là gì?**
Bộ chuyển văn bản ⇄ danh sách token id.

**13. BPE là gì?**
Byte Pair Encoding: gộp dần các cặp ký tự/byte thường gặp thành token lớn hơn cho tới khi đủ từ vựng.

**14. Encoding trong tiktoken là gì?**
Bảng từ vựng + luật gộp gắn với họ model: `o200k_base` (GPT-4o/4.1/o3...), `cl100k_base` (GPT-3.5/4 cũ).

**15. Vì sao cần ánh xạ model→encoding?**
Đếm bằng encoding sai sẽ ra số sai; mỗi model có encoding riêng.

**16. tiktoken đếm chính xác cho mọi model không?**
Chính xác cho model OpenAI; với Claude/Gemini chỉ là ước tính gần đúng.

**17. Input vs output token?**
Input = prompt gửi đi; output = văn bản model sinh ra (thường đắt hơn).

**18. Credit khác token thế nào?**
Credit là đơn vị tính phí trừu tượng của một số công cụ (như Kiro), không phải số token.

**19. Cache token và reasoning token là gì?**
Cache token = token đọc lại từ cache (rẻ); reasoning token = token "suy nghĩ" của model o-series. App có gom cả hai.

**20. FastAPI là gì?**
Web framework Python async, validate dữ liệu bằng Pydantic, tự sinh docs.

**21. Pydantic dùng để làm gì ở đây?**
Định nghĩa `CountRequest` để FastAPI tự parse + validate body POST.

**22. lru_cache làm gì?**
Nhớ kết quả `_encoder_for(model)` để khỏi tạo lại encoder mỗi lần gõ.

**23. dataclass dùng ở đâu?**
`UsageRecord`, `SourceScan` trong scanner — gọn, dễ `asdict()`.

**24. Vì sao lọc dòng bằng regex trước khi parse JSON?**
Parse JSON tốn kém; lọc nhanh giúp bỏ qua phần lớn dòng vô quan trong file log lớn.

**25. Đệ quy dùng để làm gì?**
Duyệt cây JSON lồng nhau (không biết trước độ sâu) để tìm object usage.

**26. Debounce là gì?**
Hoãn gọi hàm tới khi người dùng ngừng gõ (220ms) để tránh spam API.

---

## C. Backend chi tiết (27–44)

**27. Vì sao import tiktoken kiểu try/except?**
Để app vẫn khởi động khi thiếu tiktoken; `/api/health` báo trạng thái.

**28. Vì sao import scan_all hai kiểu?**
Để chạy được dù uvicorn gọi từ gốc (`backend.main:app`) hay trong backend/ (`main:app`).

**29. `_encoder_for` xử lý mấy trường hợp?**
3: tên encoding trực tiếp; `encoding_for_model`; fallback theo bảng → cuối cùng `o200k_base`.

**30. `_count` xử lý text None thế nào?**
`encode(text or "")` — coi None/rỗng là chuỗi rỗng.

**31. `/api/count` trả về gì?**
input_tokens, output_tokens, total_tokens, input_chars, output_chars, model, encoding.

**32. `MAX_RECORDS` và `MAX_SCAN_FILE_BYTES` để làm gì?**
Giới hạn 2000 bản ghi và bỏ qua file >80MB để tránh treo app.

**33. `scan_all` gộp dữ liệu thế nào?**
Gọi 3 scanner, `pop("_records")` gộp lại, sắp mới nhất trước, cắt 2000, tính totals.

**34. Mô tả các bước trong một scanner.**
Tìm thư mục tồn tại → duyệt file theo pattern → lọc dòng regex → parse JSON → dò usage → fingerprint chống trùng → tạo UsageRecord → set status.

**35. status của một nguồn có những giá trị nào?**
`missing` (không thấy thư mục), `no_usage` (có thư mục, không có data), `ok` (có records).

**36. `scan_kiro` xử lý gì đặc biệt?**
Đọc credit từ `meteringEvent` và % context, ghi `kind="credit_usage"` với note rõ "không có token split".

**37. `_looks_like_usage` quyết định thế nào?**
True nếu dict có bất kỳ khóa nào trong tập usage keys, hoặc có `usageMetadata` lồng cũng là usage.

**38. `_record_from_usage` chuẩn hóa ra sao?**
Dùng `_int_first` thử nhiều tên khóa cho từng loại token; total = lấy trực tiếp hoặc cộng các thành phần.

**39. `_parse_json_line` chịu lỗi thế nào?**
Thử parse cả dòng, rồi thử từ dấu `{` đầu tiên (bỏ tiền tố log); trả None nếu vẫn lỗi.

**40. Fingerprint chống trùng là gì?**
Chuỗi gồm file + request_id/timestamp + nội dung usage; đã thấy thì bỏ qua.

**41. `_clean_model` làm gì?**
Cắt model id bị lặp 2 lần trong log Kiro (`claude-...claude-...`).

**42. `_timestamp_from_path` lấy thời gian từ đâu?**
Trích mẫu `\d{8}T\d{6}` trong tên file; không có thì dùng mtime.

**43. Vì sao `_iter_matching_lines` dùng `errors="ignore"`?**
Để ký tự không decode được không làm sập việc đọc file.

**44. `_totals` tính gì?**
Tổng mọi loại token + credit (làm tròn 6 số) cho tổng toàn cục và từng nguồn.

---

## D. Frontend & câu hỏi sâu (45–60)

**45. Vì sao `escapeHtml`?**
Chống XSS — dữ liệu log không tin cậy, thoát HTML trước khi render.

**46. Debounce triển khai thế nào?**
`clearTimeout` rồi `setTimeout(fn, 220)` mỗi lần gõ.

**47. Vì sao chỉ render 250 dòng records?**
Giữ DOM nhẹ dù có hàng nghìn bản ghi.

**48. Logic parse usage có ở đâu?**
Cả backend (Python) lẫn frontend (JS) — cùng triết lý đệ quy + thử nhiều khóa.

**49. Đánh đổi của việc nhân đôi logic parse?**
Tiện cho client (phản hồi tức thì) nhưng phải đồng bộ 2 nơi khi thêm tên khóa mới.

**50. Nếu file log 5GB thì sao?**
Bị bỏ qua vì vượt 80MB — đánh đổi an toàn/đầy đủ; có thể nâng giới hạn hoặc đọc theo luồng nếu cần.

**51. Nếu schema log của công cụ thay đổi?**
Chỉ cần thêm tên khóa vào tập usage keys; thiết kế dò-theo-khóa giúp dễ mở rộng.

**52. Rủi ro quyền riêng tư?**
Đọc log có thể chạm dữ liệu nhạy cảm; app giảm thiểu bằng chỉ-đọc, chỉ lấy metadata, không gửi mạng.

**53. Vì sao không lưu kết quả quét?**
Thiết kế stateless: mỗi lần quét đọc lại từ log gốc → luôn mới, không tạo bản sao dữ liệu.

**54. Làm sao thêm hỗ trợ một công cụ AI mới?**
Viết thêm một hàm `scan_xxx()` theo cùng khuôn (tìm thư mục → lọc → parse → dò usage), thêm vào `scan_all`.

**55. Vì sao đếm token ở backend chứ không client?**
tiktoken là thư viện Python; để đếm chuẩn cần chạy server-side. (Client chỉ parse JSON usage có sẵn.)

**56. Hạn chế lớn nhất của app?**
Phụ thuộc việc công cụ có ghi metadata xuống local không; nhiều công cụ chỉ lộ credit hoặc không ghi gì.

**57. Cải tiến tiếp theo?**
Thêm biểu đồ theo thời gian, ước tính chi phí (token × đơn giá), hỗ trợ thêm nguồn, xuất CSV.

**58. Vì sao tách generic encoding (`o200k_base`) trong danh sách model?**
Cho người dùng chọn thẳng encoding khi muốn đếm theo bảng cụ thể, không qua tên model.

**59. Điều gì xảy ra nếu JSON dán vào không có usage?**
`findUsage` trả null → hiển thị lỗi "No usage object found", không crash.

**60. Bài học lớn nhất từ dự án?**
Xử lý dữ liệu thực tế đòi hỏi phòng thủ kỹ và trung thực: thà báo "không có dữ liệu" còn hơn đưa con số sai.

---

## Mẹo trả lời
- Nhấn 2 trụ cột: **đếm token chính xác** (tiktoken+cache) và **quét log phòng thủ + trung thực**.
- Khi nói về log, luôn kèm ví dụ cụ thể (Kiro chỉ có credit → ghi chú rõ).
- Thể hiện tư duy bảo mật: local-only, chỉ-đọc, escapeHtml.
