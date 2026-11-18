# 01 — Tổng quan dự án

## 1. Dự án này là gì? (cho người chưa biết gì)

Khi bạn dùng các công cụ lập trình AI (Claude Code, Kiro, Google Antigravity, hay gọi API OpenAI...), mỗi lần trò chuyện đều tiêu tốn **token** — đơn vị tính chi phí của mô hình ngôn ngữ. Token nhiều = tiền nhiều. Nhưng các công cụ thường **không hiển thị rõ** bạn đã dùng bao nhiêu.

**AI Usage Monitor** là một **web app chạy ngay trên máy bạn** giúp trả lời câu hỏi "tôi đã dùng bao nhiêu token/credit?". Nó có 2 chế độ:

1. **Quét log local (Actual usage scanner)**: tự tìm và đọc các file log mà những công cụ AI ghi xuống máy bạn, trích ra metadata về token/credit *khi có*.
2. **Ước tính thủ công (Manual tokenizer estimate)**: bạn dán văn bản (prompt + câu trả lời) vào, app dùng `tiktoken` đếm số token chính xác theo từng model.

Ngoài ra có ô **dán JSON usage** từ response API để lấy con số usage thật mà nhà cung cấp trả về.

## 2. Nó giải quyết vấn đề gì?

| Vấn đề | Cách xử lý |
|--------|-----------|
| Không biết đã tiêu bao nhiêu token/credit | Quét log local của các công cụ AI, tổng hợp lại |
| Muốn ước tính token TRƯỚC khi gọi API (để dự trù chi phí) | Đếm bằng `tiktoken` theo đúng encoding của model |
| Nhà cung cấp chỉ lộ credit, không lộ token | App **trung thực** báo "không có token, chỉ có credit" thay vì bịa |
| Lo ngại quyền riêng tư | Quét **chỉ-đọc, hoàn toàn local**, không gửi dữ liệu đi đâu |

## 3. Công nghệ chính

| Thành phần | Công nghệ | Vai trò |
|-----------|-----------|---------|
| Backend | **FastAPI** + `uvicorn` | API đếm token + quét log |
| Đếm token | **tiktoken** (OpenAI) | Tokenizer BPE chính xác theo model |
| Kiểm tra dữ liệu vào | **Pydantic** (`BaseModel`) | Validate body request |
| Quét log | Thư viện chuẩn Python (`pathlib`, `re`, `json`, `dataclasses`) | Tìm/đọc/parse file log |
| Frontend | HTML/CSS/JS thuần | Bảng usage, ô đếm token, parse JSON |

## 4. Cấu trúc thư mục

```
token-counter/
├── backend/
│   ├── main.py            # FastAPI: /api/count, /api/models, /api/usage/scan, /api/health
│   └── usage_scanner.py   # Quét log Claude Code / Kiro / Antigravity
├── frontend/
│   ├── index.html         # Giao diện: metric, sources, bảng records, ô đếm token
│   ├── app.js             # Gọi API, render, parse JSON usage, debounce
│   └── style.css
├── requirements.txt       # fastapi, uvicorn, tiktoken
├── start_token_counter.bat
└── start_usage_monitor.bat
```

## 5. Các nguồn log được hỗ trợ

| Nguồn | Đọc được gì từ local |
|-------|----------------------|
| **Claude Code** | Token usage thật nếu có file JSONL/telemetry chứa metadata `usage` |
| **Kiro** | Credit đã dùng + % context từ log local (KHÔNG có split input/output token) |
| **Google Antigravity** | Token thật chỉ khi log chứa metadata; nếu không, báo "không tìm thấy" |
| **API JSON** (dán tay) | Usage thật nếu response có object `usage` |

> Triết lý quan trọng: nếu một nhà cung cấp chỉ lộ quota/credit mà không ghi token input/output xuống local, **app không giả vờ có token chính xác**. Đây là điểm trung thực được nhấn mạnh trong README.

## 6. Điểm khác biệt đáng kể

- **Quét log "phòng thủ"**: lọc dòng theo regex trước khi parse, giới hạn dung lượng file (80MB), bỏ qua file lỗi, chống trùng bằng fingerprint.
- **Dò object `usage` đệ quy**: log mỗi công cụ có cấu trúc JSON khác nhau, app duyệt cây JSON tìm bất kỳ object nào "trông giống usage" với hàng loạt tên khóa khác nhau (`input_tokens`, `prompt_tokens`, `promptTokenCount`...).
- **Hoàn toàn local, chỉ-đọc**: tôn trọng quyền riêng tư — không gửi gì ra ngoài.
- **Đếm token đúng encoding**: ánh xạ model → encoding (`o200k_base`, `cl100k_base`), cache encoder bằng `lru_cache`.

## 7. Pitch 30 giây

> "AI Usage Monitor là web app local giúp tôi biết đã tiêu bao nhiêu token/credit khi dùng các công cụ AI. Backend FastAPI có hai phần: một bộ quét log local an toàn — lọc theo regex, giới hạn dung lượng, dò object usage đệ quy qua nhiều schema khác nhau — và một bộ đếm token bằng tiktoken theo đúng encoding từng model. Triết lý xuyên suốt là trung thực: nếu nhà cung cấp chỉ lộ credit thì tôi báo credit, không bịa token."
