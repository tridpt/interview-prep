# 1. Tổng quan dự án Word Checker

## 1.1. Word Checker là gì?

Word Checker là một **công cụ rà soát file Word (.docx)** trước khi nộp — dành cho sinh viên, nhân viên văn phòng cần đảm bảo bài/báo cáo đúng quy chuẩn định dạng và sạch lỗi. Bạn đưa vào một file `.docx`, công cụ trả về danh sách lỗi: sai font/cỡ chữ, giãn dòng/canh lề/lề trang không đúng, dấu cách đôi, thiếu dấu cách sau dấu phẩy, lặp từ, thiếu viết hoa đầu câu, lỗi chính tả, tiêu đề nhảy cấp, vượt số từ... kèm thống kê (số từ, số trang ước tính, thời gian đọc).

### Điểm cốt lõi cần nhớ
- **Chạy offline** (trừ tính năng AI tùy chọn). Dữ liệu không gửi đi đâu.
- Lõi Python + **python-docx** đọc cấu trúc file Word.
- Kiến trúc **tách lớp sạch**: mỗi loại kiểm tra một module, mọi lỗi là một dataclass `Issue` chung.
- Hai cách dùng: **CLI** (dòng lệnh) và **web app kéo-thả (Flask)**.
- Không chỉ phát hiện mà còn **tự sửa** (`--fix`/`--fix-format`), **chú thích vào file** (`--annotate`), **học quy chuẩn từ file mẫu** (`--template`).
- Đóng gói được thành **`.exe`** (PyInstaller) để người không có Python vẫn dùng.

## 1.2. Dự án giải quyết vấn đề gì?

1. **Nộp bài bị trừ điểm vì lỗi định dạng/chính tả** mà mắt thường khó soi hết → tự động rà toàn bộ.
2. **Mỗi trường/công ty một quy chuẩn khác nhau** → có nhiều **profile** sẵn + **học quy chuẩn từ file mẫu**.
3. **Sửa tay tốn thời gian** → tự động sửa lỗi cơ học/chính tả/định dạng, lưu ra file mới (không đụng bản gốc).
4. **Người không rành kỹ thuật** → web app kéo-thả + file `.exe` chạy ngay.
5. **Trình diễn năng lực kỹ thuật**: phân giải cấu trúc OOXML phức tạp (font "hiệu lực"), thiết kế hệ thống kiểm tra mở rộng được, giảm nhiễu thông minh (bỏ qua công thức toán), tích hợp LLM có chống "ảo giác", và đóng gói phân phối — đây là phần "ăn điểm".

> 💡 **Mẹo phỏng vấn:** đây là dự án **xử lý tài liệu + kiến trúc phần mềm tốt**. Hãy nhấn vào: (1) giải bài toán khó "font hiệu lực" trong OOXML; (2) kiến trúc tách lớp khiến thêm một loại kiểm tra mới rất dễ; (3) tư duy **giảm báo nhầm (false positive)** — bỏ qua công thức toán, chỉ báo lỗi chắc chắn, chống AI bịa.

## 1.3. Các nhóm kiểm tra (kiểm tra những gì)

| Nhóm | Module | Ví dụ lỗi bắt được |
|------|--------|--------------------|
| **Định dạng** | `formatting.py` | sai font/cỡ chữ, canh lề, giãn dòng, lề trang, thụt lề dòng đầu, dùng nhiều font/cỡ không đồng nhất |
| **Văn bản (cơ học)** | `textcheck.py` | cách đôi, dấu cách thừa trước/thiếu sau dấu câu, khoảng trắng cuối đoạn/trong ngoặc, đoạn trống thừa, lặp từ, thiếu viết hoa, `--` thay gạch ngang, lẫn nháy thẳng/cong, chữ bỏ quên (TODO, Lorem ipsum...) |
| **Chính tả** | `spelling.py` | dò theo từ điển `common_typos.txt` (an toàn, ít báo nhầm) |
| **Chính tả AI** | `llm_spelling.py` | dùng LLM bắt lỗi rộng hơn (tùy chọn, có chống ảo giác) |
| **Tiêu đề** | `headings.py` | nhảy cấp tiêu đề, đánh số mục không liên tục |
| **Giới hạn** | `limits.py` | vượt/thiếu số từ, vượt số trang ước tính |

## 1.4. Công nghệ sử dụng (tech stack)

**Lõi (Python 3.10+):**
- **python-docx** `>=1.1` — đọc/ghi file `.docx`, phân giải đoạn/run/style/section.
- **Flask** `>=3.0` — web app kéo-thả.
- **fpdf2** `>=2.7` — xuất báo cáo PDF (font Unicode tiếng Việt).
- **httpx** + **python-dotenv** — tính năng chính tả AI (tùy chọn, OpenAI-compatible).
- Thư viện chuẩn: `re` (regex — trái tim của textcheck/spelling), `argparse` (CLI), `csv`, `json`, `dataclasses`, `collections.Counter`.

**Đóng gói & test:**
- **PyInstaller** — đóng gói thành `WordChecker.exe` (gói sẵn `web/` + `dictionaries/`).
- **pytest** — bộ test; **Pillow** (dev) để tạo icon/logo.
- **GitHub Actions** — CI chạy test.

> 💡 **Câu hỏi cài bẫy: "Vì sao chỉ hỗ trợ .docx mà không .doc?"** Trả lời: ".docx là định dạng Office Open XML (OOXML) mở — thực chất là một file zip chứa XML, python-docx đọc được. File .doc cũ là định dạng nhị phân đóng, cần thư viện khác/Word. Em yêu cầu lưu lại thành .docx, và CLI báo lỗi rõ nếu đưa vào .doc."

## 1.5. Cách dùng (một số lệnh tiêu biểu)

```bash
pip install -r requirements.txt

# Kiểm tra cơ bản
python cli.py bai_tap.docx

# Xuất báo cáo HTML/PDF/JSON
python cli.py bai_tap.docx --html bao_cao.html --pdf bao_cao.pdf --json kq.json

# Chọn profile, hoặc học quy chuẩn từ file mẫu
python cli.py bai_tap.docx --profile luan_van
python cli.py bai_tap.docx --template mau_chuan.docx

# Tự động sửa (không ghi đè bản gốc), sửa cả định dạng
python cli.py bai_tap.docx --fix --fix-format

# Chú thích comment vào file, kiểm tra giới hạn, bật AI
python cli.py bai_tap.docx --annotate
python cli.py bai_tap.docx --max-words 2000 --min-words 500
python cli.py bai_tap.docx --llm-spelling

# Kiểm cả thư mục, gộp 1 báo cáo
python cli.py thu_muc --html bao_cao_tong.html

# Web app kéo-thả
python webapp.py     # mở http://127.0.0.1:5000
```
Công cụ **thoát với mã 1** nếu có lỗi nghiêm trọng (tiện CI/tự động hóa), `0` nếu file sạch.

## 1.6. Cấu trúc dự án

```
word-checker/
├─ cli.py            # điểm vào dòng lệnh (argparse)
├─ webapp.py         # web app kéo-thả (Flask)
├─ launch.py         # launcher cho bản .exe (tự mở trình duyệt)
├─ build_exe.bat / WordChecker.spec  # đóng gói PyInstaller
├─ config.py         # quy chuẩn định dạng (profiles), bật/tắt từng kiểm tra
├─ web/index.html    # giao diện kéo-thả
├─ dictionaries/common_typos.txt   # từ điển lỗi chính tả
├─ tests/            # bộ test pytest
└─ checker/          # LÕI: mỗi loại kiểm tra một module
   ├─ document.py    # đọc .docx, phân giải font "hiệu lực"
   ├─ issues.py      # dataclass Issue (cấu trúc một lỗi)
   ├─ runner.py      # gom logic chạy kiểm tra (dùng chung CLI + web)
   ├─ formatting.py / textcheck.py / spelling.py / llm_spelling.py
   ├─ headings.py / limits.py / stats.py / mathdetect.py
   ├─ learn_profile.py   # học quy chuẩn từ file mẫu
   ├─ autofix.py / annotate.py    # sửa & chú thích
   ├─ report.py / pdf_report.py   # xuất HTML/PDF
   └─ resources.py   # phân giải đường dẫn (dev + .exe)
```

> ⚠️ **Lưu ý bảo mật (nên chủ động nêu):** web app **chạy cục bộ, không có xác thực**; file upload chỉ xử lý tạm rồi xóa, không lưu, không gửi đi đâu. README ghi rõ đừng mở cổng này ra Internet công cộng. Tính năng AI gửi văn bản tới dịch vụ LLM nên chỉ bật khi chấp nhận điều đó.
