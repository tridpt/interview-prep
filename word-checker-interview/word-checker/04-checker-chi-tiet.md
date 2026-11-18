# 4. Lõi `checker/` chi tiết (từng module, từng hàm)

> Viết sau khi đọc trực tiếp toàn bộ package `checker/`. Đọc kèm code thật.

---

## 4.1. `issues.py` — Cấu trúc một lỗi

Định nghĩa hằng `SEVERITY_ERROR/WARNING`, các `CATEGORY_*`, và:
```python
@dataclass
class Issue:
    category: str; severity: str; message: str
    paragraph: int | None = None; excerpt: str = ""; suggestion: str = ""
    def location(self): return "Toan tai lieu" if paragraph is None else f"Doan {paragraph}"
    def to_dict(self): ...   # cho JSON
```
**Trái tim của kiến trúc**: mọi module nói cùng "ngôn ngữ" qua `Issue`.

---

## 4.2. `document.py` — Đọc .docx & phân giải font hiệu lực (khó nhất)

- `RunInfo(text, font_name, font_size)`, `ParagraphInfo(...)`, `DocInfo(...)` — các dataclass mô tả tài liệu đã "tiêu hóa".
- **`_doc_default_font(document)`**: đọc XML thô `w:docDefaults/w:rPrDefault/w:rPr` qua `qn(...)` để lấy font/cỡ mặc định (API cấp cao không lộ). Cỡ ở half-points nên `/2`.
- **`_style_font(style)`**: duyệt chuỗi kế thừa style (`base_style`) tìm font/cỡ đầu tiên, có chống vòng lặp (`seen`).
- **`_resolve_run(run, paragraph, defaults)`** ⭐: phân giải font/cỡ **hiệu lực** theo 4 tầng: run → character style → paragraph style → docDefaults.
- **`_line_spacing(p)`**: trả giãn dòng nếu là **bội số** (float như 1.5); nếu là khoảng cách tuyệt đối (Pt) → None (bỏ qua).
- **`_para_indent_spacing(p)`**: thụt lề dòng đầu (cm), space before/after (pt).
- **`load_document(path)`** ⭐: lặp `document.paragraphs`, với mỗi đoạn dựng `ParagraphInfo` (đếm `number` = thứ tự đoạn **không rỗng**, là số người dùng thấy), gom `fonts`/`sizes` hiệu lực; đọc lề trang từ `sections[0]`. Bọc `try/except` chỗ đọc XML/lề để không sập với file lạ.

> 💡 Nói được "font hiệu lực 4 tầng" + "đếm số đoạn bỏ đoạn rỗng để khớp cảm nhận người dùng" là điểm cộng lớn.

---

## 4.3. `mathdetect.py` — Nhận diện công thức (giảm nhiễu)

`is_math_heavy(text)` trả True nếu: (1) chứa ký hiệu trong tập `_MATH_SYMBOLS` (∑∫√≤≥ chữ Hy Lạp...); (2) khớp `_EQ_LABEL` (số hiệu phương trình cuối dòng như `(4.1)`); (3) **tỷ lệ token kiểu toán ≥ 0.35** (biến 1 chữ cái, hàm `f(x)`, chỉ số `x_i`, toán tử thuần `=+-*/^()`). Dùng để loại đoạn công thức khỏi kiểm font/cỡ/dấu câu.

---

## 4.4. `formatting.py` — Kiểm tra định dạng

`check_formatting(doc, profile, skip_math)`:
- Lấy `body` = đoạn không rỗng & không phải heading; lọc tiếp `prose` = bỏ đoạn `is_math_heavy` (nếu skip_math).
- Mỗi đoạn prose chạy: `_check_font` (font không thuộc danh sách → **error**), `_check_size` (cỡ không cho phép → error), `_check_alignment` (canh lề; **bỏ qua đoạn ngắn <12 từ** vì tiêu đề/chú thích thường canh giữa có chủ ý → warning), `_check_line_spacing` (theo dung sai), `_check_indent_spacing` (thụt lề/space before-after nếu profile bật).
- `_check_consistency(prose)`: dùng `Counter` đếm font/cỡ; nếu body dùng >1 font hoặc >1 cỡ → **một** cảnh báo tổng "không đồng nhất".
- `_check_margins(doc, profile)`: so lề trang với dung sai.

> Chi tiết tinh tế: canh lề chỉ báo cho đoạn ≥12 từ → tránh báo nhầm tiêu đề/dòng ký tên.

---

## 4.5. `textcheck.py` — Lỗi cơ học văn bản (regex)

`check_text(doc, checks, max_consecutive_empty, skip_math)` lặp từng đoạn (bỏ đoạn rỗng & đoạn công thức), chạy các hàm con bật trong `config.TEXT_CHECKS`:
- `_check_double_space` (`  +`), `_check_space_before_punct` (`\s+([,.;:!?)])`), `_check_missing_space_after_punct` (`([,;:])(?=\S)` — **bỏ qua số:số** như "3,14"/"10:30"), `_check_trailing_space`, `_check_space_in_parens`.
- `_check_repeated_words`: regex backreference; **chỉ báo hư từ** trong `_REPEATABLE_TYPO_WORDS` ("các","và","the"...) → tránh báo nhầm **từ láy** ("xa xa").
- `_check_sentence_capitalization`: sau `.!?` mà chữ thường → cảnh báo; **chặn** viết tắt (`_ABBREVIATIONS`), số thứ tự ("1."), ellipsis ("...").
- `_check_double_hyphen` (`--`), `_check_placeholder` (Lorem ipsum/TODO/FIXME/`[...]`/`xxxx`).
- Cấp tài liệu: `_check_mixed_quotes` (lẫn `"` thẳng và `" "` cong), `_check_empty_paragraphs` (đếm chuỗi đoạn rỗng liên tiếp vượt ngưỡng).

> 💡 Cả module là minh hoạ tư duy **giảm false positive**: mỗi kiểm tra đều có ngoại lệ để không báo bừa.

---

## 4.6. `spelling.py` — Chính tả theo từ điển (offline)

- `load_typo_dict()`: đọc `dictionaries/common_typos.txt` (mỗi dòng `sai = đúng`, bỏ dòng `#`).
- `_build_pattern(typos)`: ghép tất cả từ sai thành một regex `(?<!\w)(...)(?!\w)`, **sắp cụm dài trước** (ưu tiên khớp cụm nhiều chữ), cờ IGNORECASE|UNICODE.
- `check_spelling(doc)`: quét từng đoạn, mỗi khớp tạo một `Issue` **error** kèm gợi ý từ đúng.
- Triết lý: **an toàn, ít báo nhầm** — chỉ báo cặp lỗi đã biết chắc.

---

## 4.7. `llm_spelling.py` — Chính tả bằng AI (tùy chọn)

- `check_spelling_llm(doc)`: gom đoạn → **chia lô** (`_chunks`, giới hạn ~3500 ký tự) → prompt yêu cầu trả JSON `{p, wrong, correct, note}`.
- `_call_llm`: gọi OpenAI-compatible qua httpx (`temperature=0`, `max_tokens=1500`), bắt lỗi → `LLMError`.
- `_extract_json_array`: bóc mảng JSON kể cả khi model bọc ```` ```json ````.
- **Chống ảo giác** ⭐: chỉ nhận lỗi nếu `wrong` **thật sự xuất hiện** trong đoạn (`if wrong not in text_by_num[num]: continue`); xếp mức **warning**.

---

## 4.8. `headings.py` — Tiêu đề & đánh số

- `_check_level_jumps`: theo dõi cấp Heading; nếu `level > prev_level + 1` (vd 1→3) → cảnh báo thiếu cấp trung gian.
- `_check_numbering_sequence`: bóc số đề mục trong text (`_NUM_RE`), kiểm con của mỗi parent có liên tục không (1,2,4 → thiếu 3); cập nhật theo số thực để **không báo dây chuyền**.

---

## 4.9. `limits.py` & `stats.py`

- `stats.compute_stats(doc)`: đếm số từ/ký tự/đoạn/tiêu đề; ước tính số trang (`350 từ/trang`) và thời gian đọc (`200 từ/phút`).
- `limits.check_limits(doc, limits)`: so số từ/trang với `min_words/max_words/max_pages` → `Issue` **error** kèm gợi ý "cắt/viết thêm bao nhiêu từ".

---

## 4.10. `learn_profile.py` — Học quy chuẩn từ file mẫu

- `learn_profile(path)`: thống kê `Counter` font/cỡ/giãn dòng/canh lề/lề/thụt lề trong thân bài file mẫu.
- `_common_values(counter, total, threshold=0.2)`: lấy các giá trị xuất hiện ≥20% (luôn gồm phổ biến nhất) → cho phép nhiều font/cỡ hợp lệ.
- `_dominant(counter)`: giá trị phổ biến nhất (cho giãn dòng/canh lề/lề).
- Trả về một **profile** y như cấu trúc trong config → kiểm các file khác theo mẫu. `describe_profile` in tóm tắt cho người dùng.

---

## 4.11. `autofix.py` — Tự động sửa (giữ định dạng)

- **Nguyên tắc an toàn**: không ghi đè bản gốc; sửa ở cấp **run** để giữ font/màu/đậm-nghiêng; chỉ sửa lỗi chắc chắn.
- `_fix_run_text(text, typos)`: bỏ dấu cách trước dấu câu, gộp cách đôi, thêm dấu cách sau `,;:`, sửa chính tả theo từ điển (**giữ kiểu viết hoa** qua `_match_case`).
- `_remove_extra_empty_paragraphs`: xóa đoạn trống thừa (thao tác XML `el.getparent().remove(el)`).
- `_fix_formatting(document, profile)` (khi `--fix-format`): chọn **font/cỡ đích = giá trị cho phép đang dùng nhiều nhất** (tránh đặt về giá trị đầu danh sách gây lệch), đặt lại canh lề/giãn dòng/thụt lề/space; `_fix_margins` đặt lề trang.
- `autofix(...)` trả thống kê số lỗi đã sửa.

> 💡 "Sửa ở cấp run để không phá định dạng" và "chọn font đích phổ biến nhất thay vì phần tử đầu danh sách" là hai chi tiết tinh tế đáng kể.

---

## 4.12. `annotate.py` — Chú thích comment vào .docx

- Tái tạo cách đánh số đoạn (`number`) giống `load_document` để ánh xạ `Issue.paragraph` về đúng đoạn.
- Gom lỗi theo đoạn; mỗi đoạn lỗi → `document.add_comment(p.runs, text=..., author="Word Checker")`. Lỗi cấp tài liệu gom vào một comment ở đoạn đầu. File gốc không đổi.

---

## 4.13. `report.py`, `pdf_report.py`, `resources.py`, `runner.py`

- `report.aggregate_issues(issues, threshold=8)` ⭐: **gộp lỗi lặp** cùng message khi >8 đoạn thành một dòng "— N đoạn (ví dụ đoạn ...)" cho gọn — **chỉ cho hiển thị**; JSON/annotate giữ nguyên đầy đủ.
- `report.print_report/write_html/write_html_multi`: in console + HTML (tô màu theo severity, bản gộp nhiều file).
- `pdf_report.write_pdf`: fpdf2 + font Unicode để in tiếng Việt đúng.
- `resources.resource_path`: phân giải đường dẫn cho cả dev (`__file__/..`) lẫn .exe (`sys._MEIPASS`).
- `runner.run_checks`: điều phối tất cả theo cờ (đã mô tả ở file 03); **import muộn** `llm_spelling` chỉ khi cần.

---

## 4.14. Bộ test (tests/)

- `conftest.py`: fixture `make_doc` tạo `.docx` tạm với định dạng cho trước → test cô lập.
- Mỗi module một file test; kiểm cả ca **đúng** lẫn ca **không báo nhầm**: `test_missing_space_not_flag_numbers` ("3,14"/"10:30" không bị báo), `test_repeated_word_ignores_reduplication` ("xa xa" không bị báo), `test_sentence_cap_ignores_abbrev_number` ("Mục 1." không bị báo thiếu viết hoa), `test_no_mixed_quotes_when_consistent`...

> 💡 Bộ test **đặc biệt nhấn vào false-positive** — thể hiện đúng triết lý của dự án.

---

## 4.15. Tổng kết quyết định kỹ thuật (lõi)

| Quyết định | Lý do |
|-----------|-------|
| `Issue` chung cho mọi module | Mọi phần xuất/hành động xử lý đồng nhất, mở rộng dễ |
| Phân giải font hiệu lực 4 tầng | Đúng như Word hiển thị, tránh báo nhầm font |
| `mathdetect` bỏ qua công thức | Giảm false positive trên tài liệu học thuật |
| Mỗi kiểm tra đều có ngoại lệ | Triết lý "thà bỏ sót hơn báo bừa" |
| Chống ảo giác AI (cụm phải có thật) | Không nhận lỗi AI bịa |
| autofix sửa ở cấp run + font đích phổ biến nhất | Giữ định dạng, không gây lệch |
| `aggregate_issues` chỉ cho hiển thị | Báo cáo gọn nhưng JSON/annotate đủ |
| runner dùng chung + import LLM muộn | DRY + không bắt buộc httpx khi không dùng |
