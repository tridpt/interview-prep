# 08 — Cheat Sheet (lướt phút chót)

## Pitch 30 giây
> AI Usage Monitor là web app local giúp biết đã tiêu bao nhiêu token/credit khi dùng công cụ AI. Backend FastAPI có 2 phần: bộ quét log local an toàn (lọc regex, giới hạn 80MB, dò usage đệ quy qua nhiều schema, chống trùng) và bộ đếm token bằng tiktoken theo đúng encoding model. Triết lý xuyên suốt: trung thực — chỉ lộ credit thì báo credit, không bịa token.

## Hai trụ cột
1. **Đếm token**: `tiktoken` + ánh xạ model→encoding + `lru_cache`.
2. **Quét usage**: đọc log Claude/Kiro/Antigravity, phòng thủ + trung thực.

## Endpoint
| Method | Path | Vai trò |
|--------|------|---------|
| GET | `/api/health` | server + có tiktoken? |
| GET | `/api/models` | danh sách model preset |
| POST | `/api/count` | đếm token input/output |
| GET | `/api/usage/scan` | quét toàn bộ log |

## Encoding
- `o200k_base`: GPT-4o, 4.1, o3, o4-mini.
- `cl100k_base`: GPT-3.5 Turbo, GPT-4 cũ.
- Fallback model lạ → `o200k_base`.

## Quy trình một scanner (nhớ nằm lòng)
tìm thư mục tồn tại → duyệt file theo pattern → **lọc dòng regex** → parse JSON (chịu lỗi) → **dò usage đệ quy** → **fingerprint chống trùng** → tạo `UsageRecord` → set status (`ok`/`no_usage`/`missing`).

## Hằng số an toàn
- `MAX_RECORDS = 2000`
- `MAX_SCAN_FILE_BYTES = 80MB`

## Các loại token gom được
input, output, cache_read, cache_creation, reasoning, total + credits + context_percent.

## Thuật ngữ 1 dòng
- **Token**: mẩu văn bản, đơn vị tính tiền.
- **BPE**: gộp cặp ký tự thường gặp thành token.
- **Encoding**: bảng từ vựng tokenizer.
- **Credit**: đơn vị phí trừu tượng (≠ token).
- **lru_cache**: nhớ encoder.
- **Debounce**: hoãn gọi tới khi ngừng gõ (220ms).
- **Fingerprint**: chuỗi định danh bản ghi để chống trùng.

## Điểm "ăn điểm"
- Quét **phòng thủ**: regex lọc, giới hạn file, errors="ignore", bắt OSError.
- Dò usage **đệ quy** phủ nhiều schema (input_tokens / promptTokenCount / candidatesTokenCount...).
- **Trung thực**: Kiro chỉ có credit → ghi note, không bịa token.
- **Privacy**: local-only, chỉ-đọc, không gửi mạng.
- **escapeHtml** chống XSS ở frontend.

## Câu "tủ" về hạn chế
> "App chỉ lấy được gì công cụ ghi xuống local. Nhiều công cụ chỉ lộ credit hoặc không ghi token, nên tôi báo đúng những gì có và ghi chú rõ — thà thiếu còn hơn sai. tiktoken cũng chỉ chính xác cho model OpenAI; với Claude/Gemini là ước tính."

## Bẫy hay gặp
- Mount `/` StaticFiles đặt **cuối** (sau /api).
- Encoding sai → đếm sai token.
- Parse JSON phải chịu lỗi (log có tiền tố trước `{`).
- Thêm tên khóa usage phải sửa **cả** backend lẫn frontend.
