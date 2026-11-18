# 03 — Kiến trúc hệ thống

## 1. Bức tranh tổng thể

```
┌─────────────────────────────┐         ┌────────────────────────────┐
│   FRONTEND (HTML/CSS/JS)     │         │   BACKEND (FastAPI)        │
│                             │  HTTP   │                            │
│  - Bảng metric tổng         │ ◄─────► │  main.py                   │
│  - Danh sách nguồn (sources)│  /api   │   /api/health              │
│  - Bảng records gần đây     │         │   /api/models              │
│  - Ô đếm token (manual)     │         │   /api/count   ──► tiktoken│
│  - Ô parse JSON usage       │         │   /api/usage/scan ─┐       │
└─────────────────────────────┘         │                    ▼       │
                                         │     usage_scanner.py       │
                                         │   scan_claude_code()       │
                                         │   scan_kiro()              │
                                         │   scan_antigravity()       │
                                         └──────────┬─────────────────┘
                                                    ▼
                                   Log local trên máy (chỉ đọc):
                                   ~/.claude, %APPDATA%/Kiro/logs, ...
```

## 2. Hai mảng chức năng độc lập

Dự án thực chất gộp **2 công cụ** vào một app, chia sẻ chung khung FastAPI + frontend:

| Mảng | API | Module | Phụ thuộc |
|------|-----|--------|-----------|
| **Đếm/ước tính token** | `/api/count`, `/api/models` | `main.py` | `tiktoken` |
| **Quét usage thật** | `/api/usage/scan` | `usage_scanner.py` | thư viện chuẩn |

Hai mảng **không phụ thuộc nhau**: nếu thiếu `tiktoken`, phần đếm báo lỗi rõ ràng nhưng phần quét vẫn chạy, và ngược lại.

## 3. Các endpoint

| Method | Đường dẫn | Vai trò | Trả về |
|--------|-----------|---------|--------|
| GET | `/api/health` | Kiểm tra server + có tiktoken không | `{status, tokenizer}` |
| GET | `/api/models` | Danh sách model preset | `{models: [...]}` |
| POST | `/api/count` | Đếm token input/output theo model | `{input_tokens, output_tokens, total, encoding...}` |
| GET | `/api/usage/scan` | Quét toàn bộ log local | `{totals, sources, records}` |
| (mount) | `/` | Phục vụ frontend tĩnh | index.html |

## 4. Luồng dữ liệu — Đếm token

```
1. Người dùng gõ vào ô Input/Output, hoặc chọn model
2. frontend debounce 220ms → POST /api/count {model, input_text, output_text}
3. main.py:
   - Pydantic parse thành CountRequest
   - _encoder_for(model): tra ENCODING_BY_MODEL, lấy encoder (cache lru)
   - _count(text): encoder.encode(text) rồi len()
4. trả {input_tokens, output_tokens, total_tokens, input_chars, output_chars, encoding}
5. frontend cập nhật các con số
```

## 5. Luồng dữ liệu — Quét usage

```
1. Người dùng bấm "Scan Local Logs" (hoặc tự chạy khi tải trang)
2. GET /api/usage/scan
3. scan_all() gọi lần lượt:
     scan_claude_code(), scan_kiro(), scan_antigravity()
   mỗi hàm:
     - tìm các thư mục log tồn tại (_existing_paths)
     - duyệt file theo pattern (_iter_files)
     - lọc dòng theo regex (_iter_matching_lines)  ← tối ưu
     - parse JSON từng dòng (_parse_json_line)
     - dò object usage (đệ quy) → tạo UsageRecord
     - chống trùng bằng fingerprint/seen set
4. scan_all() gộp records, sắp theo thời gian, cắt MAX_RECORDS (2000), tính totals
5. trả JSON → frontend render bảng nguồn + bảng records + 4 metric tổng
```

## 6. Mô hình dữ liệu (dataclass)

```python
@dataclass
class UsageRecord:        # 1 bản ghi sử dụng
    source, kind, timestamp, model, path, request_id
    input_tokens, output_tokens, cache_read_tokens,
    cache_creation_tokens, reasoning_tokens, total_tokens
    credits, context_percent, note

@dataclass
class SourceScan:         # kết quả quét 1 nguồn
    id, name, status, paths, files_scanned, records, note, totals
```
- `status` của nguồn có 3 giá trị: `missing` (không thấy thư mục), `no_usage` (có thư mục nhưng không có dữ liệu), `ok` (tìm thấy records).
- `kind`: `token_usage` (có token) hoặc `credit_usage` (chỉ có credit, như Kiro).

## 7. Cách backend nhúng frontend

```python
BASE_DIR = Path(__file__).resolve().parent.parent
FRONTEND_DIR = BASE_DIR / "frontend"
app.mount("/", StaticFiles(directory=FRONTEND_DIR, html=True))
```
Mount `/` ở **cuối file** (sau khi khai báo mọi route `/api`) để không che các API.

## 8. Import "mềm" để chịu lỗi

```python
try:
    from usage_scanner import scan_all       # chạy từ backend/
except ModuleNotFoundError:
    from backend.usage_scanner import scan_all  # chạy từ gốc dự án

try:
    import tiktoken
except ImportError:
    tiktoken = None                          # vẫn khởi động, báo qua /api/health
```
- Import 2 kiểu để chạy được dù `uvicorn` được gọi từ thư mục nào.
- `tiktoken` import "mềm": nếu thiếu thì `tiktoken = None`, app vẫn lên, phần đếm token báo lỗi 500 với hướng dẫn cài đặt.

## 9. Nguyên tắc thiết kế cốt lõi

1. **Chỉ-đọc, local-only**: scanner không bao giờ ghi/sửa/xóa log, không gửi mạng.
2. **Phòng thủ trước file thật**: log thực tế lộn xộn, to, có thể hỏng → mọi bước đều bọc try/except và giới hạn.
3. **Trung thực dữ liệu**: phân biệt rõ token (chính xác) vs credit (trừu tượng) vs ước tính (tiktoken cho model không phải OpenAI).
4. **Hai công cụ tách biệt**: lỗi ở mảng này không kéo sập mảng kia.
