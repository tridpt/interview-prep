# 02 — Khái niệm nền tảng (từ con số 0)

---

## 1. Token là gì?

Mô hình ngôn ngữ (LLM) không đọc văn bản theo từng chữ cái hay từng từ, mà theo **token** — những "mẩu" văn bản. Một token có thể là một từ ngắn, một phần của từ, một dấu câu, hay khoảng trắng. Ví dụ tiếng Anh: trung bình ~4 ký tự ≈ 1 token; "ChatGPT" có thể tách thành "Chat" + "GPT".

**Vì sao quan trọng?** Vì các nhà cung cấp **tính tiền theo token** (cả token đầu vào lẫn đầu ra). Biết số token = dự trù được chi phí và không bị "vỡ kế hoạch".

## 2. Tokenizer là gì?

**Tokenizer** là bộ chuyển đổi văn bản ⇄ danh sách token (thực chất là danh sách số nguyên — "token id"). Mỗi model dùng một tokenizer cụ thể; đếm token bằng tokenizer *sai* sẽ ra số *sai*.

## 3. BPE (Byte Pair Encoding)

Đa số tokenizer hiện đại dùng **BPE**. Ý tưởng: bắt đầu từ các ký tự/byte đơn lẻ, rồi **gộp dần** những cặp xuất hiện nhiều nhất thành token lớn hơn, lặp lại cho tới khi đủ kích thước từ vựng. Kết quả: từ thông dụng thành 1 token, từ hiếm bị tách nhỏ. BPE cân bằng giữa "từ vựng gọn" và "biểu diễn được mọi văn bản".

## 4. tiktoken và "encoding"

**tiktoken** là thư viện tokenizer BPE của OpenAI, rất nhanh. Mỗi họ model gắn với một **encoding** (bảng từ vựng + luật gộp):

| Encoding | Dùng cho model |
|----------|----------------|
| `o200k_base` | GPT-4o, GPT-4.1, o3, o4-mini... (mới) |
| `cl100k_base` | GPT-3.5 Turbo, GPT-4 (cũ hơn) |
| `p50k_base`, `r50k_base` | các model GPT-3 đời cũ |

Trong dự án, mỗi preset model khai báo `encoding` tương ứng; nếu model lạ thì fallback về `o200k_base`.

> 🔑 Lưu ý: tiktoken là tokenizer của **OpenAI**. Với model của hãng khác (Claude, Gemini), số token tiktoken chỉ là **ước tính gần đúng**, không hoàn toàn chính xác. Đây là điểm cần nói thẳng khi phỏng vấn.

## 5. Token đầu vào / đầu ra (input/output tokens)

- **Input tokens** (prompt): văn bản bạn gửi cho model (câu hỏi, system message, ngữ cảnh).
- **Output tokens** (completion): văn bản model sinh ra.
- Thường giá **output đắt hơn input**. Tổng chi phí ≈ input×giá_in + output×giá_out.
- Một số nhà cung cấp còn tách **cache tokens** (token đọc từ cache, rẻ hơn) và **reasoning tokens** (token "suy nghĩ" của model o-series). Dự án có gom cả các loại này.

## 6. Credit vs Token

Không phải công cụ nào cũng lộ token. Một số (như Kiro trên máy thử nghiệm) chỉ ghi **credit** (đơn vị tính phí trừu tượng) và **% context đã dùng**. App tôn trọng điều đó: báo credit, không suy ra token giả.

## 7. FastAPI — framework backend

**FastAPI** là web framework Python hiện đại:
- Khai báo route bằng decorator: `@app.get("/api/health")`, `@app.post("/api/count")`.
- **Tự validate** body request bằng **Pydantic** (`BaseModel`) → nếu client gửi sai kiểu, FastAPI trả lỗi 422 tự động.
- Hỗ trợ async, tự sinh tài liệu OpenAPI.
- Phục vụ file tĩnh qua `StaticFiles`.

## 8. Pydantic — kiểm tra dữ liệu vào

```python
class CountRequest(BaseModel):
    model: str = "gpt-4o-mini"
    input_text: str = ""
    output_text: str = ""
```
Khi client POST `/api/count`, FastAPI tự parse JSON thành đối tượng `CountRequest`, đảm bảo đúng kiểu và điền giá trị mặc định nếu thiếu. → Code an toàn, ít boilerplate.

## 9. `lru_cache` — bộ nhớ đệm cho hàm

`tiktoken.get_encoding(...)` hơi tốn thời gian. Dự án bọc hàm lấy encoder bằng `@lru_cache(maxsize=32)`: lần đầu gọi với một model thì tính và **nhớ kết quả**; các lần sau cùng model đó trả về ngay. → Tăng tốc rõ rệt khi gõ liên tục.

## 10. `dataclass` — gọn nhẹ cho cấu trúc dữ liệu

`usage_scanner.py` dùng `@dataclass` để định nghĩa `UsageRecord` và `SourceScan` — tự sinh `__init__`, dễ chuyển sang dict bằng `asdict()`. Đây là cách Python gọn để mô tả "bản ghi dữ liệu".

## 11. Regex (biểu thức chính quy) — lọc dòng log

Trước khi parse JSON (tốn kém), app dùng regex lọc nhanh: chỉ những dòng **có chứa** từ khóa như `usage`, `input_tokens`, `meteringEvent`... mới được xét tiếp. Đây là tối ưu quan trọng vì file log có thể rất to.

## 12. Đệ quy (recursion) — duyệt cây JSON

Mỗi công cụ AI ghi log với cấu trúc JSON lồng nhau khác nhau. App dùng hàm **đệ quy** `walk()` để đi sâu vào mọi dict/list, tìm bất kỳ object nào "trông giống usage". Đệ quy là kỹ thuật tự nhiên để duyệt cấu trúc cây không biết trước độ sâu.

## 13. Debounce — chống gọi API liên tục

Ở frontend, khi người dùng gõ vào ô văn bản, nếu mỗi phím gõ đều gọi API thì quá tải. **Debounce**: chờ một khoảng ngắn (220ms) sau lần gõ cuối rồi mới gọi. Triển khai bằng `clearTimeout` + `setTimeout`.

---

## Bảng thuật ngữ nhanh

| Thuật ngữ | Một câu |
|-----------|---------|
| Token | Mẩu văn bản — đơn vị model đọc và tính tiền |
| Tokenizer | Bộ chuyển văn bản ⇄ token id |
| BPE | Thuật toán gộp cặp ký tự thường gặp thành token |
| Encoding | Bảng từ vựng tokenizer (o200k_base, cl100k_base) |
| Input/Output token | Token prompt gửi đi / token model sinh ra |
| Credit | Đơn vị tính phí trừu tượng (không phải token) |
| FastAPI | Web framework Python async |
| Pydantic | Validate dữ liệu vào bằng class |
| lru_cache | Nhớ kết quả hàm để khỏi tính lại |
| dataclass | Cách gọn định nghĩa bản ghi dữ liệu |
| Debounce | Hoãn gọi hàm tới khi ngừng gõ |
