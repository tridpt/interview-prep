# 5. Giao diện & xuất kết quả chi tiết (CLI, Web app, .exe)

> Hai giao diện dùng chung lõi `runner.run_checks`. Đọc kèm `cli.py`, `webapp.py`, `launch.py`.

---

## 5.1. `cli.py` — Giao diện dòng lệnh (argparse)

**Khởi đầu:** ép `stdout/stderr` về UTF-8 (tránh lỗi font tiếng Việt trên console Windows cp1252).

**Các hàm:**
- `_gather_docx(path)`: nếu là thư mục → liệt kê mọi `.docx` (bỏ file tạm `~$`); nếu là file → trả một phần tử.
- `_check_one(path, profile, args)`: gọi `run_checks` với các cờ `--no-*` đảo lại.
- `_limits_from_args`: lấy giới hạn từ CLI, thiếu thì dùng `config.LIMITS`.
- `_suffix_path(src, suffix)`: dựng tên file ra (vd `bai_fixed.docx`).

**Hàm `run(args)` — luồng chính:**
1. Kiểm file tồn tại; lấy `profile` (theo `--profile` hoặc **học từ `--template`** qua `learn_profile`, in mô tả profile đã học).
2. Nếu `--llm-spelling` mà chưa cấu hình key → lỗi 2.
3. Gom danh sách `.docx` hợp lệ (báo lỗi nếu đưa `.doc`).
4. **Chế độ sửa** (`--fix`/`--fix-format`): chỉ cho **một file** (an toàn/dễ hiểu), gọi `autofix` → in thống kê → **kiểm lại file đã sửa** để báo phần còn sót.
5. Chạy kiểm + `compute_stats` cho từng file; `print_report` (một file) hoặc in từng file + tổng kết (nhiều file).
6. Xuất theo cờ: `--html` (đơn/`_multi`), `--pdf` (file đầu), `--json` (`to_dict`), `--annotate` (mỗi file → `*_commented.docx`).
7. **Exit code**: trả `1` nếu có bất kỳ `Issue` severity error, ngược lại `0`.

**`main()`:** định nghĩa toàn bộ cờ argparse (`--profile/--template/--fix/--fix-format/--max-words/--min-words/--max-pages/--out/--html/--pdf/--json/--annotate/--no-format/--no-text/--no-headings/--no-skip-math/--no-spelling/--llm-spelling`).

> 💡 Điểm hay: `--fix` rồi **kiểm lại bản đã sửa** để cho người dùng thấy còn sót gì; exit code 1/0 cho CI.

---

## 5.2. `webapp.py` — Web app kéo-thả (Flask)

**Khởi tạo:** `Flask` với `template_folder`/`static_folder` qua `resource_path(...)` (để chạy được cả khi đóng gói .exe). Giới hạn upload **25MB** (`MAX_CONTENT_LENGTH`). Chỉ chấp nhận `.docx`.

**Helper:**
- `_save_upload(file_storage)`: lưu upload ra **file tạm** (`tempfile.mkstemp`).
- `_resolve_profile(req)`: nếu có file `template` upload → `learn_profile`; ngược lại theo tên profile trong form.
- `_valid_upload(req)`: kiểm có file + đuôi `.docx`.

**Các route:**
- `GET /` → render `index.html` (truyền danh sách profile, profile mặc định, `llm_enabled`).
- `POST /check` → lưu tạm → `run_checks` (theo các cờ trong form) + `compute_stats` → trả JSON `{filename, summary, stats, issues}` (issues đã **aggregate** để hiển thị) → **xóa file tạm** trong `finally`.
- `POST /fix` → `autofix` → `send_file(..._fixed.docx)`.
- `POST /annotate` → `run_checks` + `annotate` → `send_file(..._commented.docx)`.
- `POST /pdf` → `run_checks` + `write_pdf` → `send_file(..._bao_cao.pdf)`.
- `GET /favicon.ico` → trả icon.

**Bảo mật/riêng tư:** mọi route lưu file ra **tạm** rồi **xóa ngay** (`finally: os.remove`); chạy `host="127.0.0.1"` (chỉ localhost); **không có xác thực** → không mở ra Internet.

> 💡 Câu hỏi hay: "Upload file có an toàn không?" → File chỉ xử lý tạm rồi xóa, không lưu/không gửi đi đâu, giới hạn 25MB, chỉ localhost. Hạn chế: không auth → chỉ dùng cục bộ.

---

## 5.3. `launch.py` + đóng gói `.exe`

- `launch.py`: launcher cho bản đóng gói — `_free_port()` chọn cổng rảnh (ưu tiên 5000), mở trình duyệt sau 1.2s (thread), rồi `app.run`.
- `build_exe.bat` + `WordChecker.spec`: PyInstaller gói app + **kèm `web/` và `dictionaries/`** thành `dist/WordChecker.exe`. Double-click là chạy server cục bộ + mở trình duyệt; copy mỗi file `.exe` sang máy Windows khác là dùng được.
- `resources.py` đảm bảo đường dẫn tài nguyên đúng khi chạy từ `.exe` (`sys._MEIPASS`).

---

## 5.4. `web/index.html` — Giao diện

Trang kéo-thả: chọn/kéo một hoặc nhiều `.docx`, chọn quy chuẩn (hoặc upload file mẫu), bấm **Kiểm tra** (gọi `/check`, hiển thị báo cáo ngay), hoặc **Tự động sửa & tải về** (`/fix`), **Tải file có chú thích** (`/annotate`), **Tải PDF** (`/pdf`). Có nút **sáng/tối**.

---

## 5.5. Xuất kết quả (tóm tắt — chi tiết ở file 04)

- **Console**: `report.print_report` — thống kê + danh sách lỗi đã gộp, dấu `[X]` (error) / `[!]` (warning).
- **HTML**: `report.write_html(_multi)` — bảng tô màu, bản gộp nhiều file.
- **PDF**: `pdf_report.write_pdf` — fpdf2 + font Unicode tiếng Việt.
- **JSON**: list `{file, issues:[issue.to_dict()]}` cho tự động hóa.

---

## 5.6. Tổng kết quyết định kỹ thuật (giao diện)

| Quyết định | Lý do |
|-----------|-------|
| CLI + web dùng chung `run_checks` | Không lặp logic (DRY) |
| `--fix` rồi kiểm lại bản đã sửa | Cho người dùng thấy phần còn sót |
| Exit code 1/0 | Tích hợp CI/tự động hóa |
| Web lưu file tạm rồi xóa ngay | Riêng tư; không lưu dữ liệu người dùng |
| `resource_path` cho dev + .exe | Cùng code chạy ở cả hai môi trường |
| Đóng gói .exe gói sẵn web+dict | Người không có Python vẫn dùng |
| UTF-8 reconfigure stdout | In tiếng Việt đúng trên Windows |
