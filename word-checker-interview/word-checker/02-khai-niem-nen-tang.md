# 2. Các khái niệm nền tảng (lý thuyết)

File này giải thích **mọi thuật ngữ** trong dự án, cho người chưa biết gì. Nắm chắc phần này sẽ giúp bạn tự tin trước mọi câu hỏi "khái niệm".

---

## 2.1. File .docx & OOXML

**`.docx` là gì?** Là định dạng **Office Open XML (OOXML)** — bản chất là một file **ZIP** chứa nhiều file XML mô tả nội dung, định dạng, style... (đổi đuôi `.docx` thành `.zip` rồi giải nén sẽ thấy). Khác với `.doc` cũ (định dạng nhị phân đóng).

**Các khái niệm cấu trúc Word cần biết:**
- **Paragraph (đoạn)**: một khối văn bản. Mỗi đoạn có **style**, canh lề, giãn dòng, thụt lề...
- **Run**: một "mẩu" văn bản **cùng định dạng** bên trong một đoạn. Một đoạn có thể có nhiều run (ví dụ một câu có vài chữ in đậm → chữ đậm là một run riêng). Font/cỡ chữ gắn ở cấp run.
- **Style**: bộ định dạng đặt tên (Normal, Heading 1, Title...). Style có thể **kế thừa** (based_on) style khác.
- **Section**: phần tài liệu chứa thiết lập trang (lề trang, khổ giấy).
- **docDefaults**: định dạng mặc định toàn tài liệu (font/cỡ mặc định).

---

## 2.2. python-docx

Thư viện Python đọc/ghi `.docx`. Trong dự án:
- `Document(path)` mở file; `document.paragraphs` (danh sách đoạn), `p.runs`, `p.style`, `p.alignment`, `p.paragraph_format` (line_spacing, first_line_indent, space_before/after), `document.sections[0]` (lề trang).
- Đơn vị: python-docx trả về `Length` (Pt, Cm...) — code dùng `.pt`, `.cm` để đổi.
- Ghi: đặt `run.font.name`, `run.font.size = Pt(...)`, `p.alignment`, `section.top_margin = Cm(...)` rồi `document.save(out)`.
- Truy cập XML thô qua `document.styles.element` + `qn("w:...")` (namespace) để đọc `docDefaults` mà API cấp cao không lộ ra.

---

## 2.3. "Font hiệu lực" (effective font) — bài toán khó nhất

**Vấn đề:** một run **có thể không khai báo font/cỡ trực tiếp**. Khi đó Word lấy font theo thứ tự ưu tiên:
1. Font đặt thẳng trên **run**.
2. Nếu không có → từ **character style** của run.
3. Nếu vẫn không → từ **paragraph style** (đi theo chuỗi `based_on`).
4. Cuối cùng → **docDefaults** (mặc định tài liệu).

**Giải pháp trong `document.py`:** hàm `_resolve_run` thử lần lượt 4 nguồn trên; `_style_font` duyệt chuỗi style kế thừa (`based_on`) để tìm giá trị đầu tiên; `_doc_default_font` đọc XML `docDefaults`. → Mỗi run ra một font/cỡ **hiệu lực** chính xác như Word thực sự hiển thị.

> 💡 Đây là chi tiết kỹ thuật "ăn điểm" nhất khi nói về dự án: "Em không chỉ đọc font trực tiếp trên run mà phân giải đúng font *hiệu lực* qua 4 tầng kế thừa của OOXML — nếu không sẽ báo nhầm hàng loạt."

---

## 2.4. Regex (biểu thức chính quy) — công cụ chính của kiểm tra văn bản

`textcheck.py` và `spelling.py` dùng nhiều **regex** (`import re`) để tìm mẫu lỗi:
- `r"  +"` → nhiều dấu cách liên tiếp (cách đôi).
- `r"\s+([,.;:!?)])"` → dấu cách thừa trước dấu câu.
- `r"([,;:])(?=\S)"` → thiếu dấu cách sau dấu câu (dùng **lookahead** `(?=...)` để không "ăn" ký tự sau).
- `r"(?<![^\W\d_])([^\W\d_]+)(\s+)(\1)(?![^\W\d_])"` → lặp từ liền nhau (dùng **backreference** `\1` và **lookbehind/lookahead** chặn ranh giới).
- Chính tả: ghép tất cả từ sai thành một pattern `(?<!\w)(a|b|c)(?!\w)`, **sắp cụm dài trước** để ưu tiên khớp cụm nhiều chữ.

**Khái niệm regex cần biết:** lookahead `(?=)`, lookbehind `(?<=)`, negative `(?!)`/`(?<!)`, backreference `\1`, cờ `IGNORECASE`/`UNICODE`. Đây là phần dễ bị hỏi sâu.

---

## 2.5. Profile quy chuẩn & "học từ file mẫu"

- **Profile**: một dict mô tả quy chuẩn (font cho phép, cỡ chữ, giãn dòng, canh lề, lề trang, dung sai...). `config.PROFILES` có sẵn `hanh_chinh`, `luan_van`, `cong_ty`.
- **Dung sai (tolerance)**: vì số đo thực tế hiếm khi tròn (giãn dòng 1.49 vs 1.5, lề 2.98cm vs 3.0cm), mỗi kiểm tra có một **dung sai** để không báo nhầm.
- **Học từ file mẫu** (`learn_profile.py`): đưa một `.docx` đúng chuẩn, công cụ **thống kê** font/cỡ/giãn dòng/canh lề phổ biến nhất trong thân bài (dùng `Counter`, ngưỡng xuất hiện ≥ 20%) rồi tạo ra một profile → kiểm các file khác theo đúng mẫu đó. Đây là cách "không cần sửa config bằng tay".

---

## 2.6. Nhận diện công thức toán (giảm nhiễu)

**Vấn đề:** tài liệu học thuật nhiều công thức (dùng font/cỡ riêng, nhiều ký hiệu lạ, nhiều dấu cách quanh ký hiệu) → dễ bị **báo nhầm** lỗi font/dấu câu.

**Giải pháp `mathdetect.py`:** hàm `is_math_heavy(text)` coi một đoạn là "công thức" nếu: (1) chứa ký hiệu toán mạnh (∑∫√≤≥α…), hoặc (2) có số hiệu phương trình cuối dòng `(4.1)`, hoặc (3) **tỷ lệ token kiểu toán** (biến một chữ cái, hàm `f(x)`, chỉ số `x_i`, toán tử `=+-*/^`) ≥ 35%. Các kiểm tra font/cỡ/dấu câu **bỏ qua** các đoạn này (bật/tắt bằng `--no-skip-math`).

> 💡 Đây là ví dụ rất tốt về tư duy **giảm false positive** — một công cụ báo nhầm nhiều sẽ bị người dùng tắt đi.

---

## 2.7. Dataclass Issue & mức độ/nhóm

- Mọi lỗi là một **`@dataclass Issue`** thống nhất: `category`, `severity`, `message`, `paragraph`, `excerpt` (trích đoạn), `suggestion` (gợi ý sửa).
- **severity**: `error` (gần chắc sai, nên sửa) hoặc `warning` (nên xem lại).
- **category**: Định dạng / Văn bản / Chính tả / Chính tả (AI) / Tiêu đề / Giới hạn.
- Một cấu trúc chung giúp mọi module "nói cùng ngôn ngữ", và phần xuất (HTML/PDF/JSON/annotate) xử lý đồng nhất.

---

## 2.8. LLM cho chính tả (tùy chọn) & chống "ảo giác"

- `llm_spelling.py` gọi LLM theo chuẩn **OpenAI-compatible** (Groq/Gemini/OpenAI) qua `httpx`, yêu cầu trả **JSON** danh sách lỗi.
- **Chia lô (chunk)** theo số ký tự để prompt không quá dài; `temperature=0` cho ổn định.
- **Chống ảo giác (hallucination)**: chỉ nhận một lỗi nếu **cụm từ sai thật sự xuất hiện** trong đoạn (`if wrong not in text_by_num[num]: continue`) → AI bịa thì bị loại. Lỗi AI xếp mức **warning** (chỉ gợi ý).
- TẮT mặc định; chỉ bật khi có `LLM_API_KEY`.

---

## 2.9. PyInstaller & đóng gói .exe

- **PyInstaller** đóng gói app Python + phụ thuộc thành một `.exe` chạy không cần cài Python.
- Khi chạy từ `.exe`, file dữ liệu (`web/`, `dictionaries/`) được giải nén vào thư mục tạm `sys._MEIPASS`. Vì vậy có `resources.py` → `resource_path()` phân giải đường dẫn đúng cho **cả khi dev lẫn khi đã đóng gói** (kiểm `sys.frozen`).
- `launch.py` là launcher cho `.exe`: tự chọn cổng rảnh, mở trình duyệt, chạy Flask.

---

## 2.10. pytest & fixture

- Test dùng **pytest** với **fixture** `make_doc` (trong `conftest.py`): trả về một hàm `build(paragraphs)` tạo file `.docx` tạm với định dạng cho trước (font/cỡ/canh lề/giãn dòng) trong `tmp_path` → test **cô lập**, không cần file thật.
- Mỗi module checker có một file test tương ứng (`test_textcheck.py`, `test_formatting.py`, ...), kiểm cả ca **đúng** (bắt được lỗi) lẫn ca **không báo nhầm** (vd "3,14" không bị báo thiếu dấu cách, từ láy "xa xa" không bị báo lặp từ).

---

## 2.11. Thuật ngữ khác

- **Run**: mẩu văn bản cùng định dạng (xem 2.1).
- **False positive (báo nhầm)**: báo lỗi ở chỗ thực ra đúng — dự án rất chú trọng giảm cái này.
- **Aggregate (gộp lỗi)**: khi cùng một lỗi lặp ở nhiều đoạn (>8), báo cáo gộp thành một dòng "— N đoạn" cho gọn (chỉ ở phần hiển thị; JSON/annotate giữ nguyên đầy đủ).
- **Exit code**: mã thoát chương trình; 1 nếu có lỗi nghiêm trọng, 0 nếu sạch — tiện cho tự động hóa/CI.
- **OpenAI-compatible**: định dạng API chuẩn OpenAI mà Groq/Gemini cũng theo → đổi nhà cung cấp chỉ cần đổi base_url/key/model.
