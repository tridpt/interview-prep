# 2. Kể chuyện theo STAR (câu hỏi hành vi)

Câu hỏi hành vi ("Kể về lúc em gặp một bug khó", "Lúc em phải tự học công nghệ mới") nên trả lời theo **STAR**:

- **S (Situation)**: bối cảnh ngắn gọn.
- **T (Task)**: nhiệm vụ/vấn đề của bạn.
- **A (Action)**: bạn đã LÀM gì (phần dài nhất — cụ thể, có kỹ thuật).
- **R (Result)**: kết quả (có số liệu nếu được) + bài học.

> Mẹo: dành ~70% thời gian cho **A**. Người phỏng vấn muốn nghe *bạn* làm gì, không phải "team đã làm".

---

## 8 câu chuyện làm sẵn (rút từ các dự án)

### 1. "Bug/thử thách kỹ thuật khó nhất" — Font hiệu lực (word-checker)
- **S**: Em làm công cụ kiểm tra định dạng file Word.
- **T**: Báo lỗi font sai, nhưng hầu hết đoạn lại bị báo nhầm hoặc bỏ sót.
- **A**: Em phát hiện run trong .docx thường **không khai báo font trực tiếp** — Word lấy theo 4 tầng kế thừa (run → character style → paragraph style theo based_on → docDefaults). Em viết hàm phân giải "font hiệu lực" duyệt đủ 4 tầng, đọc cả XML thô docDefaults qua namespace.
- **R**: Hết báo nhầm hàng loạt; em hiểu sâu cấu trúc OOXML. Bài học: phải hiểu đúng mô hình dữ liệu trước khi viết kiểm tra.

### 2. "Tối ưu/xử lý ràng buộc" — Lưu trữ 2 tầng (quan-quen-map)
- **S/T**: App bản đồ lưu ảnh quán; localStorage chỉ ~5MB, ảnh base64 làm đầy ngay.
- **A**: Em tách ảnh sang **IndexedDB** (dung lượng lớn, async), localStorage chỉ giữ `{id, category}`; khi khởi động nạp ảnh vào cache rồi "hydrate" lại, và dọn ảnh mồ côi.
- **R**: Không còn đầy localStorage; có test kiểm chứng đúng kiến trúc. Bài học: chọn đúng kho cho đúng loại dữ liệu.

### 3. "Đảm bảo chất lượng" — Test cho code mật mã (totp-manager)
- **S/T**: Tự cài thuật toán TOTP — sai một bit là mã vô dụng, rất khó tin "chạy được là đúng".
- **A**: Em kiểm chứng bằng **test vector chính thức của RFC 6238** (SHA1/256/512) và dùng **property-based testing (hypothesis)** cho các bất biến (vd số giây còn lại luôn trong [1, period]).
- **R**: Tin chắc implement đúng chuẩn. Bài học: với thuật toán, dùng chuẩn chính thức + fuzzing mạnh hơn vài ca thủ công.

### 4. "Giảm phiền nhiễu/nghĩ cho người dùng" — Giảm false positive (word-checker)
- **A**: Em thêm hàng loạt ngoại lệ có chủ đích: bỏ qua đoạn công thức toán, "3,14" không báo thiếu dấu cách, từ láy "xa xa" không báo lặp từ; và viết **test riêng cho các ca "không được báo nhầm"**.
- **R**: Công cụ đáng tin, người dùng không tắt đi. Bài học: công cụ báo nhầm nhiều sẽ bị bỏ — "thà bỏ sót hơn báo bừa".

### 5. "Học công nghệ mới nhanh" — OAuth/Graph/IMAP (outlook-mail-reader)
- **A**: Em phải hiểu device-code flow của MSAL, refresh token xoay vòng, và fallback sang IMAP XOAUTH2 khi Graph thiếu quyền; xử lý import nhiều tài khoản bằng job đa luồng có hủy được.
- **R**: App đọc mail đa tài khoản chạy ổn. Bài học: chia nhỏ vấn đề (auth → đọc → hiển thị) và luôn có đường lui (fallback).

### 6. "Thiết kế trải nghiệm trong tình huống khẩn" — Panic Mode (cap-cuu-101)
- **A**: App sơ cứu có chế độ đọc to từng bước (Web Speech); em xử lý getVoices bất đồng bộ, cảnh báo khi thiếu giọng tiếng Việt, và mẹo viết "một một năm" để máy đọc đúng "115".
- **R**: Dùng được rảnh tay lúc hoảng loạn. Bài học: nghĩ cho ngữ cảnh thật của người dùng, không chỉ trường hợp lý tưởng.

### 7. "Cải tiến dựa trên đo lường" — Nhiễu bền EoT (art-shield)
- **S**: Em làm công cụ thêm nhiễu đối kháng bảo vệ tranh khỏi việc bị AI cào đi train.
- **T**: Nhiễu PGD ngây thơ trông "chạy được" nhưng bị xóa sạch chỉ sau khi nén/resize ảnh — sống sót **0/9** phép tẩy nhiễu.
- **A**: Em xác định gốc rễ: nhiễu nằm ở tần số cao, bị bộ lọc làm mượt xóa mất. Em áp **EoT (Expectation over Transformation)**: mỗi vòng tối ưu, áp nhiều phép biến đổi khả vi ngẫu nhiên (mờ/resize/nén) rồi lấy trung bình gradient — kèm mẹo **STE** cho gradient chảy qua phép làm tròn (nén). Thêm **perceptual masking** giấu nhiễu vào vùng nhiều chi tiết.
- **R**: Độ bền nhảy từ **0/9 lên 9/9**, SSIM đạt 0.94 (nhiễu gần vô hình). Bài học: phải **đo bằng tiêu chí đúng** (cosine với ảnh gốc, không phải "đảo nhãn") và cải tiến dựa trên số liệu, không phải cảm tính.

### 8. "Trung thực với dữ liệu / xử lý dữ liệu bẩn" — Quét log AI (token-counter)
- **S/T**: Em làm app đo token/credit dùng AI bằng cách đọc log local của Claude Code/Kiro/Antigravity — mỗi công cụ ghi JSON cấu trúc và tên khóa khác nhau, file to và có thể hỏng.
- **A**: Em viết bộ quét **phòng thủ**: lọc dòng bằng regex trước khi parse, bỏ file >80MB, đọc với `errors='ignore'`, chống trùng bằng fingerprint; và dò object usage **đệ quy** qua nhiều schema (`input_tokens`/`promptTokenCount`/`candidatesTokenCount`...). Quan trọng nhất: khi Kiro chỉ lộ **credit** chứ không có token, em báo đúng credit kèm ghi chú, **không bịa** token.
- **R**: App chạy ổn trên log thực tế lộn xộn, số liệu đáng tin. Bài học: với dữ liệu thật phải phòng thủ kỹ, và **thà báo "không có dữ liệu" còn hơn đưa con số sai**.

---

## Cách dùng
- Khi được hỏi câu hành vi, chọn câu chuyện hợp nhất, kể theo S-T-**A**-R.
- Một câu chuyện có thể tái dùng cho nhiều câu hỏi (bug khó / tự hào nhất / học nhanh).
- Luôn kết bằng **bài học** — cho thấy bạn rút kinh nghiệm.

> 💡 Học thuộc 3 câu chuyện mạnh nhất (1, 3, 4) đủ phủ phần lớn câu hỏi hành vi. Nếu phỏng vấn thiên AI/ML, ưu tiên câu **7** (cải tiến dựa trên đo lường) và **8** (trung thực dữ liệu).
