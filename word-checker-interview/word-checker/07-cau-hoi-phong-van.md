# 7. Ngân hàng câu hỏi phỏng vấn & trả lời mẫu

70+ câu, chia nhóm, từ dễ đến khó. Mỗi câu có gợi ý trả lời. Đọc rồi tự diễn đạt lại.

> ⭐ = câu rất hay được hỏi.

---

## A. Tổng quan & động lực

**A1. ⭐ Giới thiệu ngắn gọn về dự án.**
> Word Checker là công cụ chạy offline rà soát file .docx trước khi nộp: kiểm định dạng (font/cỡ/giãn dòng/canh lề/lề trang), lỗi cơ học văn bản, chính tả (từ điển + tùy chọn AI), tiêu đề/đánh số, giới hạn số từ/trang. Lõi Python + python-docx, kiến trúc tách lớp (mỗi kiểm tra một module, lỗi là Issue chung), dùng qua CLI hoặc web app Flask, có tự sửa/chú thích/học từ mẫu, đóng gói được thành .exe. Có bộ test pytest + CI.

**A2. Vì sao làm dự án này?**
> Nộp bài hay bị trừ điểm vì lỗi định dạng/chính tả mà soi tay khó hết. Em muốn một công cụ tự rà, sửa được, và phổ cập tới người không rành kỹ thuật (web + .exe).

**A3. Đối tượng người dùng?**
> Sinh viên, nhân viên văn phòng cần đảm bảo tài liệu đúng quy chuẩn trường/công ty trước khi nộp.

**A4. ⭐ Vì sao chỉ .docx, không .doc?**
> .docx là OOXML (zip chứa XML) mở, python-docx đọc được. .doc cũ là nhị phân đóng, cần Word/thư viện khác. Em yêu cầu lưu thành .docx và báo lỗi rõ nếu đưa .doc.

---

## B. Đọc tài liệu & OOXML

**B1. ⭐ .docx được lưu thế nào?**
> Là Office Open XML — một file ZIP chứa các file XML mô tả nội dung/định dạng/style. python-docx mở và cung cấp API cấp cao (paragraphs, runs, styles, sections).

**B2. Paragraph vs Run khác nhau?**
> Paragraph là một đoạn; Run là một mẩu văn bản cùng định dạng trong đoạn. Font/cỡ gắn ở cấp run nên một đoạn có nhiều run với định dạng khác nhau.

**B3. ⭐ "Font hiệu lực" là gì? Vì sao phải phân giải?**
> Là font Word thực sự dùng cho một run. Run có thể không khai báo font; Word lấy theo thứ tự: run → character style → paragraph style (theo based_on) → docDefaults. Nếu chỉ đọc run.font.name sẽ ra None hàng loạt và báo nhầm. Em phân giải đủ 4 tầng trong _resolve_run.

**B4. Vì sao phải đọc XML thô cho docDefaults?**
> API cấp cao của python-docx không lộ docDefaults; em dùng document.styles.element + qn("w:docDefaults/...") với namespace để đọc font/cỡ mặc định (cỡ ở half-points nên chia 2).

**B5. line_spacing xử lý thế nào?**
> Nếu là bội số (1.5) python-docx trả float → dùng được; nếu là khoảng cách tuyệt đối (Pt) → trả None để bỏ qua (không so sánh nhầm đơn vị).

**B6. "number" của đoạn là gì?**
> Số thứ tự đoạn KHÔNG rỗng — để hiển thị "Đoạn 5" khớp cảm nhận người dùng (không đếm đoạn trống).

---

## C. Các kiểm tra & giảm false positive

**C1. ⭐ Kiểm tra văn bản dùng gì? Cho ví dụ regex.**
> Dùng regex: "  +" (cách đôi), "\s+([,.;:!?)])" (cách thừa trước dấu), "([,;:])(?=\S)" (thiếu cách sau dấu, dùng lookahead), backreference cho lặp từ. Mỗi loại có ngoại lệ để không báo nhầm.

**C2. ⭐ Làm sao tránh báo nhầm "3,14" là thiếu dấu cách?**
> Kiểm ký tự trước và sau dấu: nếu cả hai là chữ số (số:số) thì bỏ qua — bao gồm 3,14 và 10:30.

**C3. ⭐ Lặp từ: sao không báo "xa xa" (từ láy)?**
> Chỉ báo khi từ lặp nằm trong danh sách hư từ (_REPEATABLE_TYPO_WORDS như "các","và","the") — những từ hầu như không lặp hợp lệ. Từ láy không nằm trong đó nên không bị báo.

**C4. Thiếu viết hoa đầu câu: chặn nhầm thế nào?**
> Bỏ qua khi token trước dấu là viết tắt (_ABBREVIATIONS), số thứ tự ("1."), hoặc ellipsis ("..."). Ví dụ "Mục 1. nội dung" không bị báo.

**C5. ⭐ Vì sao bỏ qua đoạn công thức toán?**
> Công thức dùng font/cỡ riêng, nhiều ký hiệu và dấu cách lạ → dễ báo nhầm. mathdetect.is_math_heavy phát hiện qua ký hiệu toán / số hiệu phương trình / tỷ lệ token kiểu toán ≥35%, rồi loại khỏi kiểm font/cỡ/dấu câu. Tắt bằng --no-skip-math.

**C6. Canh lề: vì sao bỏ qua đoạn ngắn?**
> Đoạn <12 từ thường là tiêu đề/chú thích/dòng ký tên, canh giữa/phải có chủ ý — không phải lỗi. Chỉ kiểm canh lề cho đoạn đủ dài.

**C7. Triết lý chung về báo lỗi của bạn là gì?**
> Thà bỏ sót hơn báo bừa (giảm false positive). Công cụ báo nhầm nhiều sẽ bị tắt. Vì vậy mỗi kiểm tra đều có ngoại lệ, và có bộ test riêng cho các ca "không được báo".

---

## D. Chính tả & AI

**D1. Chính tả offline hoạt động thế nào?**
> Đọc từ điển common_typos.txt (cặp sai=đúng), ghép thành một regex (?<!\w)(...)(?!\w), sắp cụm dài trước; mỗi khớp tạo Issue error kèm gợi ý. An toàn vì chỉ báo cặp đã biết chắc.

**D2. ⭐ Chính tả AI chống "ảo giác" thế nào?**
> Gọi LLM trả JSON danh sách lỗi, nhưng chỉ nhận một lỗi nếu cụm sai THẬT SỰ xuất hiện trong đoạn (wrong in text). AI bịa cụm không có thật → bị loại. Lỗi AI xếp mức warning (gợi ý).

**D3. Vì sao chia lô (chunk) khi gọi AI?**
> Để prompt không quá dài (giới hạn ~3500 ký tự/lô), tránh vượt context và giảm chi phí. temperature=0 cho ổn định.

**D4. AI có bắt buộc không?**
> Không, tắt mặc định; chỉ bật khi cấu hình LLM_API_KEY. Không có key thì phần còn lại vẫn chạy offline. Import llm_spelling muộn để không bắt buộc httpx khi không dùng.

**D5. Mở rộng từ điển chính tả thế nào?**
> Thêm dòng "sai = đúng" vào dictionaries/common_typos.txt — không cần sửa code.

---

## E. Profile & học từ mẫu

**E1. Profile là gì? Có mấy cái sẵn?**
> Một dict quy chuẩn (font/cỡ/giãn dòng/canh lề/lề/dung sai). Có hanh_chinh (mặc định), luan_van, cong_ty; chọn bằng --profile.

**E2. ⭐ --template học quy chuẩn thế nào?**
> learn_profile thống kê Counter font/cỡ/giãn dòng/canh lề/lề trong thân bài file mẫu; lấy giá trị xuất hiện ≥20% (cho phép nhiều giá trị) và giá trị phổ biến nhất → tạo profile y như config rồi kiểm các file khác theo mẫu.

**E3. Vì sao có dung sai (tolerance)?**
> Số đo thực hiếm khi tròn (giãn 1.49, lề 2.98cm). Dung sai tránh báo nhầm khi lệch không đáng kể.

---

## F. Tự sửa & chú thích

**F1. ⭐ --fix sửa gì? Có an toàn không?**
> Sửa cách đôi, dấu cách thừa/thiếu quanh dấu câu, khoảng trắng cuối đoạn, đoạn trống thừa, chính tả từ điển. An toàn: KHÔNG ghi đè bản gốc (lưu *_fixed.docx), sửa ở cấp run để giữ định dạng, chỉ sửa lỗi chắc chắn.

**F2. ⭐ Sửa văn bản mà không phá định dạng kiểu gì?**
> Sửa text trong từng run (run = mẩu cùng định dạng) nên font/màu/đậm-nghiêng được giữ. Chính tả giữ kiểu viết hoa qua _match_case.

**F3. --fix-format chọn font đích thế nào?**
> Chọn font/cỡ = giá trị CHO PHÉP đang dùng nhiều nhất trong body (Counter), không phải phần tử đầu danh sách — tránh đổi cả tài liệu sang giá trị gây lệch.

**F4. --annotate làm gì?**
> Tạo bản *_commented.docx: mỗi đoạn lỗi gắn một comment (document.add_comment), lỗi cấp tài liệu gom vào comment đầu. Mở bằng Word thấy ở lề phải. File gốc không đổi.

---

## G. Kiến trúc & giao diện

**G1. ⭐ Mô tả kiến trúc.**
> Hai giao diện (CLI argparse, web Flask) dùng chung runner.run_checks. document.py đọc .docx → DocInfo; các module checker trả [Issue] đồng nhất; report/pdf xuất kết quả; autofix/annotate hành động. Mọi lỗi là dataclass Issue.

**G2. ⭐ Thêm một loại kiểm tra mới cần gì?**
> Viết một hàm check_xxx(doc,...) -> [Issue], nối vào runner.run_checks (kèm cờ bật/tắt). Mọi phần xuất/hành động tự hoạt động vì dùng chung Issue. Kiến trúc plugin-like.

**G3. CLI và web có lặp logic không?**
> Không — cùng gọi run_checks. DRY. Khác nhau chỉ ở phần nhận đầu vào và xuất kết quả.

**G4. Web app có an toàn không?**
> File upload lưu tạm rồi xóa ngay (finally), không lưu/không gửi đi đâu, giới hạn 25MB, chỉ localhost. Hạn chế: không có xác thực → chỉ dùng cục bộ, đừng mở ra Internet.

**G5. ⭐ Đóng gói .exe thế nào? resources.py để làm gì?**
> PyInstaller gói app + web/ + dictionaries/ thành WordChecker.exe. Khi chạy từ .exe, dữ liệu nằm ở thư mục tạm sys._MEIPASS; resource_path() kiểm sys.frozen để trả đường dẫn đúng cho cả dev lẫn .exe. launch.py tự chọn cổng + mở trình duyệt.

**G6. Vì sao reconfigure stdout về UTF-8 trong cli?**
> Console Windows mặc định cp1252 sẽ lỗi khi in tiếng Việt; ép UTF-8 (errors="replace") để hiển thị đúng.

---

## H. Testing

**H1. ⭐ Bạn test thế nào?**
> pytest với fixture make_doc tạo .docx tạm có định dạng cho trước (font/cỡ/canh lề/giãn dòng) → test cô lập. Mỗi module checker một file test.

**H2. ⭐ Test có gì đặc biệt?**
> Nhấn vào chống false positive: test "3,14"/"10:30" không bị báo thiếu dấu cách, "xa xa" không bị báo lặp từ, "Mục 1." không bị báo thiếu viết hoa, nháy nhất quán không bị báo lẫn lộn. Kiểm cả ca đúng lẫn ca không-được-báo.

**H3. fixture make_doc làm gì?**
> Trả về hàm build(paragraphs) tạo file .docx tạm trong tmp_path với add_para (đặt font/size/align/spacing) → không cần file thật, test nhanh và độc lập.

---

## I. Câu hỏi mở / tư duy

**I1. ⭐ Thử thách khó nhất?**
> (Gợi ý) Phân giải font hiệu lực qua 4 tầng OOXML (đọc XML thô docDefaults, duyệt chuỗi style kế thừa); hoặc thiết kế hàng loạt ngoại lệ để giảm false positive mà không bỏ sót lỗi thật. Kể vấn đề → cách giải → kết quả (có test chứng minh).

**I2. ⭐ Có thêm thời gian sẽ cải thiện gì?**
> Mở rộng từ điển chính tả; hỗ trợ kiểm bảng/hình/mục lục; thêm profile theo nhiều trường; tải sẵn .exe lên GitHub Release; có thể thêm kiểm tra ngữ pháp sâu hơn; cache kết quả AI.

**I3. Giới hạn đã biết?**
> Chỉ .docx (không .doc). Chính tả từ điển hẹp (an toàn nhưng bỏ sót). AI gửi văn bản lên dịch vụ ngoài (chỉ bật khi chấp nhận). Web không có auth (chỉ local). Ước tính số trang chỉ là xấp xỉ (350 từ/trang).

**I4. Quyết định kỹ thuật tự hào nhất?**
> Phân giải font hiệu lực đúng cơ chế Word + triết lý giảm false positive có bộ test riêng — cho thấy em làm công cụ dùng được thật, hiểu sâu định dạng tài liệu.

**I5. Trade-off lớn nhất?**
> Chính tả từ điển: an toàn (ít báo nhầm) đổi lại bỏ sót lỗi ngoài từ điển → bù bằng tùy chọn AI. Bỏ qua công thức toán: giảm nhiễu nhưng có thể bỏ sót lỗi thật trong đoạn đó → cho tắt bằng --no-skip-math.

**I6. Nếu file rất lớn / nhiều file thì sao?**
> CLI kiểm cả thư mục, gộp một báo cáo. Web giới hạn 25MB. python-docx đọc trong RAM nên file cực lớn có thể chậm; có thể tối ưu bằng xử lý theo luồng nếu cần.

---

## J. Câu hỏi "code cụ thể"

**J1. aggregate_issues để làm gì?**
> Khi cùng một lỗi lặp >8 đoạn, gộp thành một dòng "— N đoạn (ví dụ đoạn ...)" cho báo cáo gọn. CHỈ cho hiển thị; JSON và annotate giữ nguyên đầy đủ.

**J2. Vì sao sắp cụm lỗi chính tả dài trước trong regex?**
> Để ưu tiên khớp cụm nhiều chữ trước cụm ngắn (tránh khớp một phần của cụm dài hơn).

**J3. _check_numbering_sequence chống báo dây chuyền thế nào?**
> Sau khi báo một mục sai (vd có 4 mà mong 3), nó cập nhật last_child theo SỐ THỰC TẾ (4), nên mục sau (5) không bị báo lệch dây chuyền.

**J4. autofix xóa đoạn trống thế nào?**
> Thao tác XML: lấy phần tử đoạn (p._element), gọi getparent().remove(el). Chỉ xóa phần vượt quá max_consecutive.

**J5. Vì sao import llm_spelling bên trong runner thay vì đầu file?**
> Import muộn: chỉ import khi do_llm_spelling=True, để người không dùng AI không bắt buộc có httpx/cấu hình, và khởi động nhanh hơn.

**J6. exit code của CLI quyết định bởi gì?**
> 1 nếu có bất kỳ Issue severity == error, ngược lại 0 — tiện cho CI/script tự động.

---

## 🎤 5 câu hay gặp nhất — luyện trước tiên
1. A1 — Giới thiệu dự án.
2. B3 — Font hiệu lực & vì sao phải phân giải.
3. C7/C5 — Triết lý giảm false positive & bỏ qua công thức.
4. G1/G2 — Kiến trúc & thêm kiểm tra mới.
5. F1/F2 — Tự sửa giữ định dạng.

## ❓ Câu hỏi NÊN HỎI LẠI nhà tuyển dụng
- Team xử lý tài liệu/định dạng (Office, PDF) thế nào, có công cụ nội bộ không?
- Quy trình giảm false positive / đánh giá chất lượng cho công cụ tự động?
- Có yêu cầu đóng gói/phân phối tới người dùng cuối (desktop) không?
