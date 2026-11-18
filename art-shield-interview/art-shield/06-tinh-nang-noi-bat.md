# 06 — Tính năng nổi bật (điểm "ăn điểm" khi kể)

File này chọn ra những điểm kỹ thuật **đáng kể nhất** để bạn nhấn mạnh khi phỏng vấn — kèm cách diễn đạt.

---

## 1. Tự cài PGD + EoT từ đầu bằng PyTorch ⭐⭐⭐

**Vì sao nổi bật**: không dùng thư viện adversarial có sẵn (như foolbox, ART) — tự viết vòng lặp gradient, phép chiếu, EoT. Chứng tỏ hiểu sâu cơ chế.

**Cách kể**: "Tôi tự tối ưu một tensor nhiễu `delta` bằng gradient của cosine similarity. Mỗi vòng tôi `backward` similarity, đi ngược dấu gradient để đẩy embedding ra xa, rồi chiếu `delta` về `[−ε, ε]` và đảm bảo ảnh vẫn trong `[0,1]`."

## 2. EoT — biến nhiễu "vô dụng" thành "bền 9/9" ⭐⭐⭐

**Vì sao nổi bật**: đây là *câu chuyện cải tiến* đẹp nhất của dự án. PGD ngây thơ sống sót 0/9 phép tẩy nhiễu; sau khi thêm EoT đạt 9/9. Có **số liệu trước/sau**.

**Cách kể**: "Nhiễu PGD thường nằm ở tần số cao, mà nén/resize xóa sạch tần số cao nên nhiễu bay mất. Tôi áp EoT: mỗi vòng tối ưu, tôi áp nhiều phép biến đổi khả vi ngẫu nhiên rồi lấy trung bình gradient, ép nhiễu phải sống sót *trong khi đang bị* mờ/resize/nén. Độ bền nhảy từ 0/9 lên 9/9."

## 3. STE — cho gradient chảy qua phép làm tròn ⭐⭐

**Vì sao nổi bật**: chi tiết kỹ thuật tinh, ít người biết. Mô phỏng nén cần làm tròn màu, mà làm tròn có gradient 0.

**Cách kể**: "Để mô phỏng nén trong EoT tôi cần lượng tử hóa màu, nhưng `round` chặn gradient. Tôi dùng straight-through estimator: forward làm tròn thật, backward coi như đồng nhất — bằng mẹo `x + (q - x).detach()`."

## 4. Perceptual masking — giấu nhiễu vào vùng nhiều chi tiết ⭐⭐

**Vì sao nổi bật**: kết hợp kiến thức **thị giác người** (contrast masking) với kỹ thuật. Có số liệu: ảnh thật đạt **SSIM 0.94** (nhiễu gần vô hình) mà vẫn bền 9/9.

**Cách kể**: "Mắt người dễ thấy nhiễu ở vùng phẳng như bầu trời, khó thấy ở vùng rậm chi tiết. Tôi tính độ lệch chuẩn cục bộ của độ sáng làm bản đồ che, rồi phân bổ ngân sách nhiễu theo đó — vùng rậm cho nhiễu mạnh, vùng trơn ghìm xuống."

## 5. Trung thực khoa học: tự đo transferability ⭐⭐⭐

**Vì sao nổi bật**: `eval_transfer.py` đo transfer trên hai biến thể CLIP: nhiễu nhắm B/32 ít làm lệch B/16 (cosine +0.84). **Không có phép đo GPT-4o** hoặc thử nghiệm huấn luyện mô hình tạo ảnh.

**Cách kể**: "Tôi không tô hồng. Tôi viết hẳn một thực nghiệm so sánh: nhiễu nhắm một model chỉ bảo vệ tốt trên chính nó, gần như vô dụng trên model khác kiến trúc. Đó là lý do không công cụ nào — kể cả Glaze — miễn nhiễm 100%. Tôi xem đây là một lớp ma sát tăng chi phí cho kẻ lạm dụng, kết hợp watermark và điều khoản sử dụng."

> 💡 Trung thực về hạn chế **ghi điểm hơn** là phóng đại. Người phỏng vấn đánh giá cao tư duy phản biện.

## 6. Ensemble attack để tăng khả năng chuyển ⭐

**Cách kể**: "Để cải thiện transfer, tôi có `pgd_protect_eot_multi` tối ưu nhiễu trên nhiều CLIP cùng lúc, ép nhiễu khai thác đặc trưng *chung* thay vì mẹo riêng của một model. Nó giúp phủ tốt cả tập model đó, dù vẫn không đảm bảo cho model lạ hoàn toàn."

## 7. Tiêu chí đánh giá đúng (cosine, không phải đảo nhãn) ⭐⭐

**Vì sao nổi bật**: thể hiện tư duy đo lường chuẩn xác.

**Cách kể**: "Ban đầu dễ tưởng 'đổi được nhãn phân loại là thành công'. Nhưng nhãn có thể không đổi dù embedding đã lệch rất xa. Nên tôi lấy **cosine similarity với ảnh gốc** làm thước đo chính, ngưỡng < 0.5; việc đảo nhãn chỉ là thông tin phụ."

## 8. Tách biến đổi train (khả vi) / test (thật) ⭐

**Cách kể**: "Tôi train EoT bằng phép biến đổi torch *khả vi* để có gradient, nhưng test độ bền bằng JPEG/resize *thật* của PIL để kết quả khách quan — không tự lừa mình bằng phiên bản xấp xỉ."

## 9. Kiến trúc lõi dùng chung CLI + Web ⭐

**Cách kể**: "`shield/pipeline.py` là lõi duy nhất; cả CLI lẫn FastAPI đều gọi nó. Đổi giao diện không đụng thuật toán."

---

## Bảng tóm tắt "kể gì theo loại JD"

| JD nhấn vào | Nhấn tính năng |
|-------------|----------------|
| AI/ML, deep learning | PGD/EoT tự cài, gradient, STE, embedding/cosine |
| Computer vision | CLIP, perceptual masking, PSNR/SSIM |
| Nghiên cứu / R&D | transferability, ensemble, trung thực về giới hạn |
| Backend/Fullstack | FastAPI, pipeline dùng chung, preset, web kéo-thả |
| Bảo mật / adversarial ML | threat model, robustness, đảo vai phòng thủ/tấn công |
