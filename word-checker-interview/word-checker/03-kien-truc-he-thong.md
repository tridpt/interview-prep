# 3. Kiến trúc hệ thống

## 3.1. Sơ đồ tổng thể

```
┌─────────────── GIAO DIỆN (2 cách dùng, cùng dùng chung lõi) ───────────────┐
│  cli.py (argparse)                 webapp.py (Flask, kéo-thả)               │
│   --html/--pdf/--json/--fix/...     /check /fix /annotate /pdf              │
│         │                                   │                              │
│         └──────────────┬────────────────────┘                              │
│                        ▼                                                    │
│              checker/runner.py  ── run_checks(path, profile, do_*) ──▶ [Issue]│
│                        │                                                    │
│   ┌────────────────────┼───────────────────────────────────────────┐       │
│   ▼          ▼         ▼          ▼          ▼          ▼            ▼       │
│ document  formatting textcheck spelling   headings   limits   llm_spelling   │
│ (đọc docx, (định dạng)(cơ học) (từ điển) (tiêu đề) (số từ/   (AI, tùy chọn) │
│  font hiệu lực)                            trang)                            │
│   │                          ▲                                              │
│   │                          │ mathdetect (bỏ qua công thức)                │
│   ▼                                                                         │
│ DocInfo (paragraphs, margins, default font)                                 │
│                                                                             │
│  Xuất kết quả: report.py (HTML/console) · pdf_report.py (PDF) ·             │
│                autofix.py (sửa) · annotate.py (comment) · JSON              │
│  Tiện ích: issues.py (Issue) · stats.py · learn_profile.py · resources.py   │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Đặc điểm cốt lõi:** **runner dùng chung** — cả CLI lẫn web app đều gọi `run_checks()`, nên logic kiểm tra **không bị lặp**. Mỗi loại kiểm tra là một module độc lập trả về danh sách `Issue` đồng nhất.

## 3.2. Phân lớp & trách nhiệm

| Lớp | File | Trách nhiệm |
|-----|------|-------------|
| Giao diện | `cli.py`, `webapp.py`, `launch.py` | Nhận đầu vào/đối số, gọi runner, xuất kết quả |
| Điều phối | `checker/runner.py` | Gom các bước kiểm tra theo cờ bật/tắt |
| Đọc tài liệu | `checker/document.py` | Đọc .docx → `DocInfo` (font hiệu lực, lề, đoạn) |
| Kiểm tra | `formatting`, `textcheck`, `spelling`, `llm_spelling`, `headings`, `limits` | Mỗi module một nhóm lỗi, trả `[Issue]` |
| Giảm nhiễu | `mathdetect.py` | Nhận diện đoạn công thức để bỏ qua |
| Cấu trúc lỗi | `issues.py` | dataclass `Issue` chung |
| Xuất kết quả | `report.py`, `pdf_report.py` | HTML/console/PDF |
| Hành động | `autofix.py`, `annotate.py` | Sửa file, chú thích comment |
| Hỗ trợ | `stats.py`, `learn_profile.py`, `resources.py` | Thống kê, học quy chuẩn, đường dẫn |
| Cấu hình | `config.py` | Profiles, bật/tắt kiểm tra, LLM |

**Điểm nói khi phỏng vấn:** kiến trúc **plugin-like** — muốn thêm một loại kiểm tra mới chỉ cần viết một hàm `check_xxx(doc, ...) -> [Issue]` và nối vào `runner.py`. Mọi thứ phía sau (xuất HTML/PDF/JSON, annotate, gộp lỗi) tự hoạt động vì dùng chung `Issue`.

## 3.3. Mô hình dữ liệu

### `DocInfo` (do `document.py` tạo ra)
```python
DocInfo(
  path, paragraphs: list[ParagraphInfo], margins_cm: dict,
  default_font, default_size
)
ParagraphInfo(
  index, number,          # number = thứ tự đoạn KHÔNG rỗng (hiển thị cho người dùng)
  text, style_name, is_heading,
  alignment, line_spacing, first_line_indent_cm, space_before_pt, space_after_pt,
  fonts: set, sizes: set, # font/cỡ HIỆU LỰC trong đoạn
  runs: list[RunInfo]
)
```
➡️ `document.py` đã "tiêu hóa" file .docx phức tạp thành cấu trúc gọn để các module kiểm tra dùng — **tách phần đọc/phân giải khỏi phần logic kiểm tra**.

### `Issue` (do mọi module kiểm tra tạo ra)
```python
Issue(category, severity, message, paragraph, excerpt, suggestion)
# severity: "error" | "warning"
# category: Định dạng | Văn bản | Chính tả | Chính tả (AI) | Tiêu đề | Giới hạn
```

## 3.4. Luồng kiểm tra (run_checks)

```
load_document(path) → DocInfo
  ├─ do_format     → check_formatting(doc, profile, skip_math)
  ├─ do_text       → check_text(doc, TEXT_CHECKS, MAX_EMPTY, skip_math)
  ├─ do_headings   → check_headings(doc, HEADING_CHECKS)
  ├─ do_spelling   → check_spelling(doc)         # từ điển
  ├─ do_llm        → check_spelling_llm(doc)      # AI (import muộn, chỉ khi bật)
  └─ always        → check_limits(doc, limits)
→ gộp tất cả thành list[Issue]
```
Lưu ý: `llm_spelling` được **import muộn** (chỉ khi `do_llm_spelling=True`) để không bắt buộc cài/đụng httpx khi không dùng.

## 3.5. Luồng các hành động đầu ra

| Hành động | File | Cách làm |
|-----------|------|----------|
| In console | `report.print_report` | Tóm tắt + danh sách lỗi (đã gộp), dùng `[X]`/`[!]` |
| HTML | `report.write_html(_multi)` | Bảng lỗi tô màu theo severity; bản gộp nhiều file |
| PDF | `pdf_report.write_pdf` | fpdf2 + font Unicode tiếng Việt |
| JSON | trong `cli.py` | `issue.to_dict()` cho tự động hóa |
| Sửa | `autofix.autofix` | Sửa ở cấp **run** để giữ định dạng; lưu file mới |
| Chú thích | `annotate.annotate` | `document.add_comment(...)` mỗi đoạn lỗi |

## 3.6. Hai giao diện, một lõi

- **CLI** (`cli.py`): `argparse` parse cờ → `run_checks` → in/xuất. Hỗ trợ kiểm cả thư mục, exit code 1/0.
- **Web app** (`webapp.py`): Flask, các route `/check`, `/fix`, `/annotate`, `/pdf`; lưu file upload ra **file tạm** rồi **xóa ngay** sau khi xử lý (`finally: os.remove`). Cùng gọi `run_checks`.
- **`.exe`** (`launch.py` + PyInstaller): chạy Flask + tự mở trình duyệt; `resources.py` lo đường dẫn tài nguyên khi đã đóng gói.

## 3.7. Vì sao kiến trúc này hợp lý?

- **DRY**: runner dùng chung → CLI và web không lặp logic.
- **Mở rộng dễ**: thêm kiểm tra = thêm một hàm trả `[Issue]`.
- **Tách quan tâm**: đọc tài liệu (document) ≠ logic kiểm tra (checkers) ≠ xuất kết quả (report/pdf) ≠ hành động (autofix/annotate).
- **An toàn dữ liệu**: offline, không ghi đè bản gốc, web xóa file tạm.
- **Phân phối dễ**: đóng gói .exe cho người không có Python.
