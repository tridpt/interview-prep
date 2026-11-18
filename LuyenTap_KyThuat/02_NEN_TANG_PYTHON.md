# 2. Nền tảng Python

Gắn với backend của các dự án (agent-talk, totp-manager, word-checker, outlook-mail-reader đều Python).

## 1. Kiểu dữ liệu & cấu trúc
- `list` / `dict` / `set` / `tuple`. `set` để khử trùng + kiểm thành viên O(1) (vd `activeTypes` set).
- List/dict comprehension: `[f(x) for x in xs if cond]`.
- `collections.Counter` (đếm tần suất — word-checker dùng để tìm font phổ biến nhất), `defaultdict`.

## 2. Hàm & tham số
- Tham số mặc định, `*args`/`**kwargs`, **keyword-only** (`def f(*, a)`).
- **Lazy import** (import bên trong hàm) — agent-talk/word-checker import LLM muộn để không bắt buộc phụ thuộc khi không dùng.

## 3. Dataclass & type hints
- `@dataclass` cho cấu trúc dữ liệu gọn (vd `Issue` trong word-checker, `ParagraphInfo`, `Session` trong totp-manager).
- Type hints (`def f(x: int) -> str`) giúp đọc code + IDE gợi ý (không bắt buộc lúc chạy).

## 4. Bất đồng bộ (async/await)
- `async def` + `await`; chạy trên event loop (agent-talk dùng FastAPI async + httpx.AsyncClient).
- **async generator**: `async def` có `yield` → stream từng mẩu (agent-talk `chat_stream` yield delta LLM).
- Khi nào async: tác vụ **I/O-bound** (gọi mạng/đọc file). CPU-bound thì async không giúp.

## 5. Xử lý lỗi & context manager
- `try/except/finally`; tạo exception riêng (vd `TOTPError`, `VaultLocked`, `WrongPassword`).
- `with open(...) as f:` (tự đóng); `with httpx.AsyncClient() as c:`.

## 6. Thư viện chuẩn hay dùng trong bộ
- `hmac`, `hashlib`, `base64`, `struct` — totp-manager tự cài TOTP.
- `re` (regex) — word-checker textcheck/spelling.
- `json`, `csv`, `pathlib.Path`, `uuid`, `os`, `threading` (outlook import job đa luồng).

## 7. Web framework
- **FastAPI** (agent-talk, totp-manager, outlook): async, Pydantic validate request, `@app.post(...)`, `StreamingResponse`, `StaticFiles`.
- **Flask** (word-checker): route đồng bộ, `send_file`, `render_template`.
- **Pydantic**: `class XxxRequest(BaseModel)` → tự validate JSON, sai kiểu trả 422.

## 8. Môi trường ảo & quản phụ thuộc
- `python -m venv .venv`, `pip install -r requirements.txt`.
- Tách `requirements.txt` (chạy) vs `requirements-dev.txt` (test/dev).

## Câu hỏi mẫu
- "list vs tuple vs set?" → đổi được/không, có thứ tự, khử trùng + tra nhanh.
- "async/await dùng khi nào?" → I/O-bound; vd gọi LLM mất vài giây.
- "Decorator là gì?" → hàm bọc hàm khác (vd `@app.post`, `@dataclass`).
- "GIL là gì?" → Global Interpreter Lock: chỉ 1 luồng chạy bytecode cùng lúc → threading hợp I/O-bound, CPU-bound nên multiprocessing.
- "Vì sao tự viết TOTP bằng hmac/hashlib?" → giảm phụ thuộc, hiểu chuẩn, kiểm bằng test vector.

> 💡 Nối nền tảng với dự án: "Em dùng async generator trong agent-talk để stream từng chữ từ LLM về trình duyệt."
