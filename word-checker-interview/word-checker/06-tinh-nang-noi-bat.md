# 6. Phân tích sâu các tính năng "ăn điểm"

Mỗi tính năng kèm: ý tưởng, cách làm, "vì sao hay".

---

## 6.1. Phân giải "font hiệu lực" trong OOXML ⭐⭐ (đặc sắc nhất)

**Ý tưởng:** báo đúng font/cỡ mà Word *thực sự hiển thị*, không chỉ font khai báo thẳng trên run.

**Cách làm:** `_resolve_run` thử 4 tầng theo đúng cơ chế OOXML: run → character style → paragraph style (đi theo `based_on`) → docDefaults (đọc XML thô). `_style_font` duyệt chuỗi kế thừa có chống vòng lặp.

**Vì sao hay:** đây là **bài toán khó nhất** của dự án. Nếu chỉ đọc `run.font.name` thì hầu hết run trả về None (vì font nằm ở style) → báo nhầm hàng loạt hoặc bỏ sót. Hiểu sâu cấu trúc Word là điểm rất ấn tượng.

**Có thể bị hỏi:** *"Vì sao phải đọc XML thô cho docDefaults?"* → vì API cấp cao của python-docx không lộ docDefaults; phải dùng `qn()` + namespace.

---

## 6.2. Triết lý giảm false positive (báo nhầm) ⭐⭐

**Ý tưởng:** một công cụ báo nhầm nhiều sẽ bị người dùng tắt. Nên **thà bỏ sót hơn báo bừa**.

**Cách làm — rất nhiều ngoại lệ có chủ đích:**
- Bỏ qua **đoạn công thức toán** (`mathdetect`) khi kiểm font/cỡ/dấu câu.
- "Thiếu dấu cách sau dấu" **bỏ qua số:số** ("3,14", "10:30").
- "Lặp từ" **chỉ báo hư từ** ("các các"), bỏ **từ láy** ("xa xa").
- "Thiếu viết hoa" **chặn** viết tắt, số thứ tự, ellipsis.
- Canh lề **bỏ qua đoạn ngắn <12 từ** (tiêu đề/chú thích canh giữa có chủ ý).
- Chính tả từ điển **chỉ báo cặp đã biết chắc**.
- Chính tả AI **chống ảo giác** (cụm phải có thật trong đoạn).
- **Bộ test riêng** kiểm các ca "không được báo nhầm".

**Vì sao hay:** thể hiện tư duy sản phẩm thực tế, không chỉ "chạy được". Đây là câu chuyện kể rất ấn tượng.

---

## 6.3. Học quy chuẩn từ file mẫu (--template) ⭐

**Ý tưởng:** không bắt người dùng sửa config — đưa một file `.docx` đúng chuẩn, công cụ tự rút ra quy chuẩn.

**Cách làm:** `learn_profile` thống kê `Counter` font/cỡ/giãn dòng/canh lề/lề/thụt lề trong thân bài; lấy giá trị xuất hiện ≥20% (cho phép nhiều giá trị hợp lệ) và giá trị phổ biến nhất → dựng một profile y như config.

**Vì sao hay:** biến một bài toán cấu hình thủ công thành "đưa mẫu là xong" — UX rất tốt, và thể hiện kỹ thuật thống kê đơn giản mà hiệu quả.

---

## 6.4. Tự động sửa giữ nguyên định dạng (--fix/--fix-format) ⭐

**Ý tưởng:** không chỉ báo lỗi mà sửa luôn, mà **không phá định dạng** sẵn có.

**Cách làm:** sửa ở **cấp run** (giữ font/màu/đậm-nghiêng của từng mẩu); chính tả **giữ kiểu viết hoa** (`_match_case`); khi sửa định dạng, chọn **font/cỡ đích = giá trị cho phép đang dùng nhiều nhất** thay vì phần tử đầu danh sách (tránh đổi cả tài liệu sang giá trị lệch). Luôn **lưu ra file mới**, không đụng bản gốc; CLI **kiểm lại bản đã sửa**.

**Vì sao hay:** sửa văn bản mà không làm hỏng định dạng là tinh tế; "chọn font đích phổ biến nhất" cho thấy nghĩ tới hệ quả thực tế.

---

## 6.5. Kiến trúc mở rộng được + một lõi cho hai giao diện ⭐

**Ý tưởng:** thêm loại kiểm tra mới dễ; CLI và web không lặp code.

**Cách làm:** mọi lỗi là `Issue` chung; mỗi kiểm tra là `check_xxx(doc,...) -> [Issue]`; `runner.run_checks` gom lại; CLI/web cùng gọi runner. Thêm kiểm tra = viết một hàm + nối vào runner, mọi phần xuất/hành động tự hoạt động.

**Vì sao hay:** thể hiện thiết kế phần mềm tốt (SOLID-ish, tách quan tâm, DRY).

---

## 6.6. Chính tả AI có chống "ảo giác" ⭐

**Ý tưởng:** dùng LLM bắt lỗi rộng hơn từ điển nhưng không để AI bịa.

**Cách làm:** chia lô, prompt trả JSON, `temperature=0`; **chỉ nhận lỗi nếu cụm sai thật sự xuất hiện trong đoạn**; xếp mức warning; tắt mặc định.

**Vì sao hay:** tích hợp LLM **có kiểm soát** — nguyên tắc vàng khi làm với AI là không tin mù kết quả.

---

## 6.7. Xuất .docx có comment tại chỗ lỗi (--annotate)

**Ý tưởng:** trả lỗi ngay trong file Word để người dùng mở bằng Word thấy comment ở lề phải.

**Cách làm:** `document.add_comment(p.runs, ...)` cho mỗi đoạn lỗi; lỗi cấp tài liệu gom vào comment ở đầu; file gốc không đổi. Cần tái tạo cách đánh số đoạn để ánh xạ đúng.

**Vì sao hay:** trải nghiệm cực thân thiện — sửa ngay trong môi trường quen thuộc (Word).

---

## 6.8. Đóng gói .exe cho người không có Python

**Ý tưởng:** phổ cập công cụ tới người không rành kỹ thuật.

**Cách làm:** PyInstaller gói app + `web/` + `dictionaries/`; `launch.py` mở trình duyệt; `resources.py` xử lý đường dẫn `sys._MEIPASS`. Double-click `.exe` là chạy.

**Vì sao hay:** nghĩ tới khâu **phân phối** — nhiều dự án bỏ quên bước này.

---

## 6.9. Bảng "tính năng ↔ khái niệm kỹ thuật" (học thuộc)

| Tính năng | Khái niệm bạn thể hiện |
|-----------|------------------------|
| Font hiệu lực | OOXML, style kế thừa, đọc XML thô, namespace qn() |
| Giảm false positive | regex nâng cao, thiết kế ngoại lệ, test chống báo nhầm |
| Học từ file mẫu | thống kê Counter, ngưỡng, sinh cấu hình |
| Autofix giữ định dạng | thao tác cấp run, giữ case, chọn giá trị đích phổ biến |
| Kiến trúc mở rộng | dataclass chung, tách quan tâm, DRY (runner dùng chung) |
| Chính tả AI | LLM OpenAI-compatible, chia lô, chống ảo giác |
| Annotate | python-docx add_comment, ánh xạ đoạn |
| Đóng gói .exe | PyInstaller, sys._MEIPASS, phân phối |
| nhận diện toán | heuristic ký hiệu/tỷ lệ token |
| stats/limits | ước tính trang/thời gian đọc |

> 💡 **Chiến lược kể chuyện:** nếu được hỏi "tự hào nhất?", chọn **font hiệu lực 4 tầng** (chiều sâu kỹ thuật xử lý tài liệu) hoặc **triết lý giảm false positive** (tư duy sản phẩm). Cả hai cho thấy bạn làm công cụ *dùng được thật*, không chỉ demo.
