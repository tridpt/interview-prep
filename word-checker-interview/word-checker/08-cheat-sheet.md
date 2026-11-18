# 8. Cheat Sheet — Xem nhanh trước giờ G

Đọc lướt 5-10 phút trước khi vào phỏng vấn.

---

## 🎯 Pitch 30 giây (học thuộc)
> Word Checker: công cụ **chạy offline** rà file `.docx` trước khi nộp — kiểm **định dạng** (font/cỡ/giãn dòng/canh lề/lề trang), **lỗi cơ học văn bản**, **chính tả** (từ điển + tùy chọn AI), **tiêu đề/đánh số**, **giới hạn số từ/trang**. Lõi **Python + python-docx**, kiến trúc tách lớp (mỗi kiểm tra một module, lỗi là `Issue` chung). Dùng qua **CLI** hoặc **web app Flask**; có **tự sửa**, **chú thích comment**, **học quy chuẩn từ file mẫu**, xuất **HTML/PDF/JSON**, đóng gói **`.exe`** (PyInstaller). Có **pytest + CI**.

## 🧱 Tech stack
- **Lõi**: Python 3.10+, python-docx, `re` (regex), argparse, dataclasses, Counter.
- **Giao diện**: Flask (web), CLI; **fpdf2** (PDF); **httpx + dotenv** (AI tùy chọn).
- **Đóng gói/test**: PyInstaller (.exe), pytest, GitHub Actions.

## 🗂️ Cấu trúc
cli.py / webapp.py / launch.py (giao diện) · config.py (profiles) · **checker/** (lõi).
checker: document (đọc+font hiệu lực) · issues (Issue) · runner (điều phối) · formatting / textcheck / spelling / llm_spelling / headings / limits / stats / mathdetect / learn_profile / autofix / annotate / report / pdf_report / resources.

## 🧩 Kiến trúc (NHỚ KỸ)
CLI + Web → **runner.run_checks(path, profile, do_*)** → các module trả **[Issue]** đồng nhất → report/pdf/json + autofix/annotate. Thêm kiểm tra = thêm 1 hàm `check_xxx(doc)->[Issue]` + nối runner.

## 📄 Đọc tài liệu (.docx = OOXML = zip XML)
**Run** = mẩu cùng định dạng; font/cỡ ở cấp run. **Font hiệu lực** (⭐): run → character style → paragraph style (based_on) → docDefaults (đọc XML thô qua qn()). `number` = thứ tự đoạn KHÔNG rỗng.

## 🔎 Các kiểm tra (regex)
double_space `  +` · space_before_punct `\s+([,.;:!?)])` · missing_space `([,;:])(?=\S)` (bỏ số:số) · repeated_words (backref, chỉ hư từ) · capitalization (chặn viết tắt/số/ellipsis) · double_hyphen · mixed_quotes · placeholder · empty_paragraphs.

## 🚫 Giảm false positive (TRIẾT LÝ)
Bỏ qua **công thức toán** (mathdetect: ký hiệu/số hiệu pt/tỷ lệ token≥35%) · "3,14"/"10:30" không báo · "xa xa" không báo lặp · "Mục 1." không báo viết hoa · canh lề bỏ đoạn <12 từ · chính tả chỉ cặp đã biết · **test riêng chống báo nhầm**.

## ✍️ Chính tả
Từ điển `common_typos.txt` (regex ghép, cụm dài trước, IGNORECASE). AI (tùy chọn): OpenAI-compatible, chia lô, temperature=0, **chống ảo giác** (cụm phải có thật), mức warning, import muộn.

## 📐 Profile & học mẫu
config.PROFILES: hanh_chinh/luan_van/cong_ty + dung sai. **--template**: learn_profile thống kê Counter (≥20%), dominant → tạo profile từ file mẫu.

## 🛠️ Autofix / Annotate
**Autofix**: sửa ở **cấp run** (giữ định dạng), giữ case (_match_case), font đích = **phổ biến nhất** trong allowed; lưu **file mới**; CLI kiểm lại bản đã sửa. **Annotate**: `add_comment` mỗi đoạn lỗi.

## 🖥️ Giao diện
CLI argparse, exit code **1/0** (CI), kiểm cả thư mục. Web Flask: /check /fix /annotate /pdf, file tạm **xóa ngay** (finally), 25MB, localhost, **không auth**. .exe: PyInstaller + launch.py + **resources.py (sys._MEIPASS)**.

## 🧪 Test
pytest + fixture **make_doc** (tạo .docx tạm). Mỗi module một file test; nhấn **ca không-được-báo-nhầm**. CI GitHub Actions.

## ⚖️ Trade-off
Chính tả từ điển: an toàn, bỏ sót → bù bằng AI. Bỏ qua công thức: giảm nhiễu, có thể sót → --no-skip-math. Chỉ .docx (không .doc).

## 🚀 Cải thiện tương lai
Mở rộng từ điển; kiểm bảng/hình/mục lục; thêm profile; GitHub Release .exe; cache AI.

## 💬 Thuật ngữ chốt
- **.docx/OOXML / paragraph / run / style / based_on / docDefaults / font hiệu lực**
- **python-docx / qn() namespace / section / line_spacing**
- **regex: lookahead/lookbehind/backreference / false positive**
- **Issue (dataclass) / severity / category / runner (DRY)**
- **mathdetect / aggregate_issues / learn_profile (Counter)**
- **autofix (cấp run) / annotate (add_comment) / exit code**
- **LLM OpenAI-compatible / chống ảo giác / chia lô**
- **PyInstaller / sys._MEIPASS / Flask / pytest fixture**

---

## ✅ Checklist trước phỏng vấn
- [ ] Pitch 30 giây.
- [ ] Giải thích font hiệu lực 4 tầng.
- [ ] Triết lý giảm false positive + ví dụ.
- [ ] Kiến trúc Issue chung + thêm kiểm tra mới.
- [ ] Autofix giữ định dạng (cấp run).
- [ ] Web app an toàn + đóng gói .exe (resources/_MEIPASS).
- [ ] 2 hạn chế + cách khắc phục.
- [ ] 2 câu hỏi ngược nhà tuyển dụng.

**Hít thở sâu. Bạn hiểu dự án này. Tự tin kể câu chuyện của mình. 🍀**
