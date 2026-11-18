# 04 — Chi tiết Backend (đọc từng hàm)

Backend gồm 2 file: `main.py` (đếm token + khung API) và `usage_scanner.py` (quét log). Đây là phần "ruột".

---

# PHẦN 1 — `main.py`

## A. Khởi tạo và import mềm
```python
try:    from usage_scanner import scan_all
except ModuleNotFoundError: from backend.usage_scanner import scan_all
try:    import tiktoken
except ImportError: tiktoken = None
```
- Import 2 kiểu để chạy được bất kể `uvicorn` gọi từ đâu (`backend.main:app` từ gốc, hay `main:app` trong backend/).
- `tiktoken` mềm: thiếu cũng không sập app.

## B. Danh sách model và ánh xạ encoding
```python
MODEL_PRESETS = [
    {"id": "gpt-4.1",  "label": "GPT-4.1", "encoding": "o200k_base"},
    {"id": "gpt-4o",   "label": "GPT-4o",  "encoding": "o200k_base"},
    {"id": "gpt-3.5-turbo", ..., "encoding": "cl100k_base"},
    {"id": "o200k_base", ...}, {"id": "cl100k_base", ...},  # encoding chung
]
ENCODING_BY_MODEL = {item["id"]: item["encoding"] for item in MODEL_PRESETS}
```
- Mỗi model gắn encoding tương ứng; có 2 mục "generic" cho phép chọn thẳng encoding.

## C. `_require_tiktoken()`
Nếu `tiktoken is None` → raise `HTTPException(500, "Missing tiktoken. Run: pip install...")`. → Báo lỗi hành động được thay vì stack trace khó hiểu.

## D. `_encoder_for(model)` — lấy encoder (có cache)
```python
@lru_cache(maxsize=32)
def _encoder_for(model):
    tk = _require_tiktoken()
    if model in {"o200k_base","cl100k_base","p50k_base","r50k_base"}:
        return tk.get_encoding(model)          # chọn encoding trực tiếp
    try:
        return tk.encoding_for_model(model)    # để tiktoken tự suy
    except KeyError:
        return tk.get_encoding(ENCODING_BY_MODEL.get(model, "o200k_base"))  # fallback
```
3 tầng xử lý: (1) nếu là tên encoding thì dùng thẳng; (2) thử để tiktoken tự ánh xạ model→encoding; (3) model lạ → fallback theo bảng, cuối cùng về `o200k_base`. `lru_cache` nhớ encoder để khỏi tạo lại.

## E. `_count(text, model)`
`return len(_encoder_for(model).encode(text or ""))` — encode rồi đếm. `text or ""` xử lý `None`/rỗng an toàn.

## F. Các endpoint
- `GET /api/health` → `{status:"ok", tokenizer: tiktoken is not None}`.
- `GET /api/models` → trả `MODEL_PRESETS`.
- `POST /api/count`: nhận `CountRequest`, đếm input + output, trả thêm `encoding`, số ký tự, tổng. Đây là API gõ-tới-đâu-đếm-tới-đó.
- `GET /api/usage/scan` → gọi thẳng `scan_all()`.

---

# PHẦN 2 — `usage_scanner.py` (phần "khó" và đáng giá nhất)

## G. Hằng số an toàn
```python
MAX_RECORDS = 2000                      # giới hạn số bản ghi trả về
MAX_SCAN_FILE_BYTES = 80 * 1024 * 1024  # bỏ qua file > 80MB
```
Tránh treo app khi gặp log khổng lồ hoặc quá nhiều bản ghi.

## H. `scan_all()` — điều phối
```python
sources = [scan_claude_code(), scan_kiro(), scan_antigravity()]
all_records = []
for source in sources:
    all_records.extend(source.pop("_records", []))   # gom records, bỏ khỏi payload
all_records.sort(key=lambda r: r.timestamp or "", reverse=True)
all_records = all_records[:MAX_RECORDS]
totals = _totals(all_records); totals["files_scanned"] = sum(...)
return {scanned_at, totals, sources, records: [asdict(r) ...]}
```
- Mỗi hàm scan trả dict có khóa tạm `_records`; `scan_all` `pop` ra để gộp rồi loại khỏi payload nguồn.
- Sắp records mới nhất trước, cắt 2000, tính tổng.

## I. `scan_claude_code()` — mẫu điển hình của một scanner
Các bước (rất đáng nhớ vì lặp lại ở cả 3 scanner):
1. **Tìm thư mục tồn tại**: `~/.claude/projects`, `%APPDATA%/Claude/local-agent-mode-sessions`.
2. **Duyệt file** `*.jsonl`, `*.log`.
3. **Lọc dòng theo regex** chứa `usage|input_tokens|output_tokens|...` (tối ưu — bỏ qua dòng vô quan).
4. **Parse JSON** từng dòng.
5. **Tìm object usage**: ưu tiên đường dẫn cụ thể `message.usage` / `usage`, không có thì dò đệ quy.
6. **Trích model, request_id, timestamp** từ nhiều vị trí có thể.
7. **Fingerprint chống trùng**: `f"{file}:{request_id or timestamp}:{json.dumps(usage)}"` — cùng bản ghi không đếm 2 lần.
8. Tạo `UsageRecord` qua `_record_from_usage(...)`.
Cuối cùng cập nhật `status` = `ok`/`no_usage`/`missing` và đính `_records`.

## J. `scan_kiro()` — xử lý "credit, không token"
- Tìm `%APPDATA%/Kiro/logs`, `~/.kiro`; duyệt file tên `5-Q Chat API*.log`.
- Lọc dòng có `meteringEvent|usageSummaryEntry|contextUsageEvent`.
- Đi vào `response.events[]`: cộng dồn `meteringEvent.usage` thành **credit**, lấy `contextUsagePercentage`, dò `modelId`.
- Nếu không thấy credit trong events, lần theo đường dẫn lồng sâu `fullMessage.message.kwargs.additional_kwargs.usageSummaryEntry`.
- Bỏ qua nếu cả credit lẫn context đều trống.
- Tạo `UsageRecord(kind="credit_usage", credits=..., context_percent=..., note="Kiro log reports credits, not exact token split.")`.
→ Đây là minh chứng cụ thể cho triết lý **trung thực**: Kiro không lộ token nên app ghi credit, kèm ghi chú rõ.

## K. `scan_antigravity()`
- Tìm `%APPDATA%/Antigravity`, `~/.antigravity`; duyệt `logs/*.log|*.jsonl` + vài file đặc biệt (`app_storage.json`, `Local State`).
- Pattern rộng (cả tên khóa kiểu Google `promptTokenCount`, `candidatesTokenCount`, `totalTokenCount`).
- Với mỗi dòng JSON, **dò TẤT CẢ** object usage (`_find_all_usage_objects`) → mỗi cái thành 1 record (chống trùng bằng fingerprint).

## L. Các hàm tiện ích quan trọng

### `_iter_matching_lines(file, pattern)` — quét an toàn
```python
if file.stat().st_size > MAX_SCAN_FILE_BYTES: return   # bỏ file quá to
regex = re.compile(pattern, re.IGNORECASE)
with file.open("r", encoding="utf-8", errors="ignore") as f:
    for line in f:
        if regex.search(line): yield line
except (OSError, UnicodeError): return                  # file lỗi → bỏ qua êm
```
3 lớp phòng thủ: giới hạn dung lượng, `errors="ignore"` cho ký tự lỗi, bắt OSError/UnicodeError.

### `_parse_json_line(line)`
Thử parse cả dòng; nếu thất bại, thử cắt từ dấu `{` đầu tiên (vì log hay có tiền tố thời gian/level trước JSON). Trả `None` nếu vẫn lỗi → không làm sập vòng lặp.

### `_find_all_usage_objects(obj)` + `_looks_like_usage(value)` — trái tim của scanner
```python
def walk(value):
    if isinstance(value, dict):
        if _looks_like_usage(value): found.append(value)
        for child in value.values(): walk(child)
    elif isinstance(value, list):
        for child in value: walk(child)
```
- **Đệ quy** duyệt mọi ngóc ngách JSON.
- `_looks_like_usage`: trả True nếu dict có **bất kỳ** khóa nào trong tập usage keys (`input_tokens`, `prompt_tokens`, `promptTokenCount`, `candidatesTokenCount`...) — phủ nhiều schema khác nhau giữa các nhà cung cấp.

### `_record_from_usage(...)` — chuẩn hóa về một schema chung
Dùng `_int_first(usage, [nhiều_tên_khóa])` để lấy token bất kể nhà cung cấp đặt tên gì:
- input: `input_tokens` / `prompt_tokens` / `inputTokenCount` / `promptTokenCount`
- output: `output_tokens` / `completion_tokens` / `candidatesTokenCount`...
- cache, reasoning tương tự.
- total: lấy `total_tokens` nếu có, không thì cộng tay các thành phần.

### `_int_first`, `_first_text`, `_get_path`, `_optional_float`, `_float`
Bộ hàm "lấy giá trị an toàn": thử nhiều khóa, ép kiểu cẩn thận (trả `None`/0 nếu lỗi), đi theo đường dẫn lồng nhau mà không crash khi khóa vắng.

### `_timestamp_from_path(path)`
Cố trích timestamp từ tên file (mẫu `\d{8}T\d{6}`); không có thì dùng thời điểm sửa file (`mtime`).

### `_clean_model(model)`
Một số log Kiro lặp model id 2 lần (`claude-...claude-...`) → hàm cắt về 1 lần. Chi tiết "dọn dữ liệu thực tế bẩn".

### `_totals(records)`
Cộng tổng mọi loại token + credit (làm tròn 6 chữ số) → dùng cho cả tổng toàn cục lẫn tổng từng nguồn.

---

## Tóm tắt tinh thần backend
> Phần đếm token thì **chính xác và đơn giản** (tiktoken + cache). Phần quét log thì **phòng thủ và linh hoạt**: lọc nhanh bằng regex, parse chịu lỗi, dò usage đệ quy qua nhiều schema, chuẩn hóa về một bản ghi chung, và trung thực phân biệt token vs credit.
