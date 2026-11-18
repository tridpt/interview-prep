# 02 — Khái niệm nền tảng (giải thích từ con số 0)

File này trang bị toàn bộ lý thuyết cần để hiểu Art Shield. Đọc kỹ phần này thì các file sau sẽ rất nhẹ nhàng.

---

## 1. Embedding (vector đặc trưng) là gì?

Máy tính không "hiểu" ảnh như người. Một mô hình AI sẽ biến ảnh thành một **dãy số** — gọi là **embedding** (hay *vector đặc trưng*). Ví dụ CLIP biến mỗi ảnh thành một vector **512 chiều**.

Ý nghĩa: hai ảnh **giống nhau về nội dung** sẽ có embedding **gần nhau** trong không gian 512 chiều; hai ảnh khác nhau thì embedding **xa nhau**. Embedding chính là "cách AI nhìn ảnh".

> 🔑 Cốt lõi của Art Shield: **làm cho embedding của ảnh bảo vệ nằm thật xa embedding của ảnh gốc**. Khi đó AI "thấy" một thứ khác hẳn dù ảnh trông y nguyên.

## 2. Cosine similarity (độ tương đồng cosine)

Làm sao đo "gần/xa" giữa hai vector? Dùng **cosine similarity** — cosin của góc giữa hai vector:

- Bằng **1.0**: hai vector cùng hướng → AI thấy "y hệt".
- Bằng **0**: vuông góc → không liên quan.
- Âm (vd **−0.4**): ngược hướng → AI thấy "trái ngược".

Trong Art Shield, embedding được **chuẩn hóa L2** (độ dài = 1), nên cosine chỉ là phép nhân chấm (dot product): `(a * b).sum()`.

> Tiêu chí "còn bảo vệ" của dự án: **cosine với ảnh gốc < 0.5**. Càng thấp càng tốt.

## 3. CLIP là gì?

**CLIP** (Contrastive Language–Image Pre-training, OpenAI 2021) là một mô hình học cách **nối ảnh với văn bản**. Nó có 2 bộ mã hóa:

- **Image encoder**: ảnh → vector 512 chiều.
- **Text encoder**: câu mô tả → vector 512 chiều.

CLIP được huấn luyện sao cho ảnh và mô tả đúng của nó có embedding gần nhau. Nhờ vậy CLIP có thể **phân loại zero-shot**: cho một ảnh và danh sách mô tả ("a landscape painting", "a portrait"...), CLIP tính cosine giữa ảnh và từng mô tả, cái nào cao nhất là nhãn dự đoán.

**Tại sao Art Shield nhắm CLIP?** Vì CLIP (và các bộ mã hóa ảnh tương tự) là "mắt" của rất nhiều hệ thống AI tạo ảnh và lọc dữ liệu. Làm hỏng đặc trưng CLIP của tranh = làm tranh khó dùng cho pipeline đó.

Art Shield dùng bản `openai/clip-vit-base-patch32` (gọi tắt B/32) làm mặc định.

## 4. Nhiễu đối kháng (adversarial perturbation) là gì?

Đây là phát hiện kinh điển của học sâu: mạng nơ-ron rất **mong manh**. Chỉ cần cộng vào ảnh một lượng nhiễu **cực nhỏ, có chủ đích** (không phải nhiễu ngẫu nhiên), ta có thể làm mô hình dự đoán sai hoàn toàn — trong khi mắt người không nhận ra khác biệt.

Lý do: mô hình là hàm số phức tạp với hàng triệu tham số. Trong không gian ảnh nhiều chiều, có những "hướng nhạy cảm" mà chỉ cần dịch chuyển pixel một chút theo đúng hướng đó, output đổi rất mạnh. Nhiễu đối kháng đi tìm chính những hướng này.

**Cách tìm hướng đó: dùng gradient.** Gradient cho biết "thay đổi pixel theo hướng nào thì hàm mục tiêu thay đổi nhanh nhất". Ta đi theo gradient để **giảm** cosine similarity giữa ảnh và bản gốc.

## 5. PGD (Projected Gradient Descent) — thuật toán tạo nhiễu

PGD là cách tạo nhiễu đối kháng phổ biến và mạnh. Quy trình (chính là `pgd_protect` trong dự án):

1. Tính embedding của ảnh gốc → `target_emb` (mục tiêu cần né xa).
2. Khởi tạo nhiễu `delta` ngẫu nhiên nhỏ trong `[−ε, ε]`.
3. Lặp `steps` vòng:
   - Tạo ảnh tạm `adv = clamp(image + delta, 0, 1)`.
   - Tính `similarity = cosine(encode(adv), target_emb)`.
   - Lan truyền ngược (`backward`) để có gradient của similarity theo `delta`.
   - **Cập nhật**: `delta -= step_size * sign(gradient)` — đi ngược dấu để **giảm** similarity (đẩy ra xa).
   - **Chiếu (project)**: cắt `delta` về lại `[−ε, ε]`, và đảm bảo `image + delta` vẫn nằm trong `[0,1]`.

Hai khái niệm cần nhớ:
- **ε (epsilon)**: biên độ nhiễu tối đa theo chuẩn **L-infinity** (mỗi pixel chỉ được đổi tối đa ε). ε nhỏ = khó thấy nhưng yếu; ε lớn = mạnh nhưng dễ lộ.
- **sign(gradient)**: PGD dùng *dấu* của gradient (FGSM-style), không dùng độ lớn — giúp bước đi đều và ổn định.

## 6. Vấn đề chí mạng: nhiễu ngây thơ bị "tẩy" dễ dàng

Nhiễu PGD thường tập trung ở **tần số cao** (thay đổi pixel li ti). Mà các thao tác đời thường — **nén JPEG, resize, làm mờ** — đều là bộ lọc làm mượt, **xóa sạch tần số cao**. Kết quả: kẻ thu thập chỉ cần resize ảnh là nhiễu bay mất, AI nhìn lại đúng tranh gốc.

Thực nghiệm trong README: PGD ngây thơ (ε=0.03) **sống sót 0/9** phép tẩy nhiễu. Vô dụng ngoài đời thực.

## 7. EoT (Expectation over Transformation) — làm nhiễu BỀN VỮNG

Giải pháp: **ép nhiễu phải sống sót *trong khi đang bị* biến đổi**. Đây là EoT (hàm `pgd_protect_eot`).

Ý tưởng: mỗi vòng lặp PGD, thay vì tính gradient trên ảnh "sạch", ta:
1. Áp **nhiều phép biến đổi ngẫu nhiên** (mờ, resize, nén, thêm nhiễu) lên ảnh `adv`.
2. Tính similarity trên **từng** ảnh đã biến đổi, rồi **lấy trung bình** (đây là "kỳ vọng" — expectation).
3. Tối ưu nhiễu dựa trên trung bình đó.

Hệ quả: nhiễu buộc phải khai thác **đặc trưng bền** — thứ vẫn còn sau khi ảnh bị xử lý. Thực nghiệm: EoT đạt **9/9** sống sót.

> 🔑 Điều kiện kỹ thuật quan trọng: các phép biến đổi dùng trong EoT phải **khả vi (differentiable)** để gradient chảy qua được. Đó là lý do dự án có 2 file riêng: `diff_transforms.py` (khả vi, bằng torch — dùng KHI TRAIN nhiễu) và `transforms.py` (bằng PIL, không khả vi — chỉ dùng để TEST).

## 8. STE (Straight-Through Estimator) — mẹo cho phép làm tròn

Khi mô phỏng nén ảnh, ta cần **lượng tử hóa màu** (làm tròn giá trị pixel). Nhưng phép làm tròn có gradient = 0 ở mọi nơi → gradient không chảy qua được, EoT vô hiệu.

Mẹo **STE**: ở chiều xuôi (forward) thì làm tròn thật; ở chiều ngược (backward) thì coi như hàm đồng nhất (gradient = 1). Trong code:
```python
q = torch.round(x * levels) / levels   # giá trị đã làm tròn
return x + (q - x).detach()            # forward = q, backward chảy qua x
```
`.detach()` cắt gradient của phần `(q - x)`, nên đạo hàm chỉ còn theo `x`.

## 9. Perceptual masking — giấu nhiễu cho đỡ lộ

Mắt người **không nhạy đều** với nhiễu: rất dễ thấy nhiễu ở **vùng phẳng, mịn** (bầu trời, tường trơn), nhưng khó thấy ở **vùng nhiều chi tiết/cạnh** (lá cây, tóc, texture). Hiện tượng này gọi là *contrast masking*.

`perceptual.py` khai thác điều đó: tính **độ lệch chuẩn cục bộ** của độ sáng (vùng nào "rậm chi tiết" thì std cao) → tạo một **bản đồ che (mask)** giá trị trong `[m_min, 1]`. Vùng rậm cho nhiễu mạnh (gần 1), vùng trơn ghìm nhiễu xuống (gần `m_min`). Nhờ vậy ảnh đạt SSIM cao hơn (nhiễu vô hình hơn) mà vẫn bền.

## 10. PSNR và SSIM — đo "ảnh có đổi với mắt người không"

Để chứng minh nhiễu khó thấy, dự án dùng 2 thước đo:

- **PSNR** (Peak Signal-to-Noise Ratio, đơn vị dB): so sánh sai khác pixel. Càng **cao** càng giống. > 40 dB là khác biệt rất khó thấy; ~30 dB vẫn khá kín.
- **SSIM** (Structural Similarity Index, 0..1): mô phỏng cách mắt người cảm nhận cấu trúc/độ tương phản. Càng gần **1** càng giống. SSIM 0.94 = nhiễu gần như vô hình.

## 11. Transferability — nhiễu có "chuyển" sang model khác không?

Câu hỏi sống còn: nhiễu nhắm CLIP-B/32 có làm hỏng được **mô hình khác** (B/16, GPT-4o...) không? Đây gọi là **transferability** (khả năng chuyển).

Trong thử nghiệm `eval_transfer.py`, khả năng chuyển giữa **hai CLIP** kém: nhiễu nhắm B/32 làm lệch embedding trên B/32 (cosine −0.49) nhưng ít làm lệch B/16 (cosine +0.84). **Chưa thử GPT-4o**, không suy diễn kết quả định lượng cho mô hình đó.

**Cách cải thiện**: tấn công **ensemble** nhiều model cùng lúc (`pgd_protect_eot_multi`) → nhiễu buộc khai thác đặc trưng chung của nhiều model, tăng phần nào khả năng chuyển. Nhưng **không bao giờ** phủ được mọi model, đặc biệt model đóng kín.

## 12. Mô hình mối đe dọa (threat model) — ai là "kẻ tấn công"?

Trong bài toán này, vai trò bị **đảo ngược** so với adversarial attack thông thường:
- "Người phòng thủ" = **họa sĩ** (tạo nhiễu để bảo vệ).
- "Kẻ tấn công" = **người cào dữ liệu** (cố tẩy nhiễu để dùng tranh train AI).

Họa sĩ thắng nếu nhiễu sống sót qua các phép tẩy. Đây là tư duy **adversarial machine learning** đặt trong bối cảnh bảo vệ bản quyền.

---

## Bảng thuật ngữ nhanh

| Thuật ngữ | Một câu |
|-----------|---------|
| Embedding | Vector số mô tả nội dung ảnh (CLIP: 512 chiều) |
| Cosine similarity | Đo độ giống hướng giữa 2 vector (1=giống, <0.5=đã bảo vệ) |
| CLIP | Mô hình nối ảnh–văn bản, "mắt AI" mà ta đánh lừa |
| Adversarial perturbation | Nhiễu nhỏ có chủ đích làm AI nhìn sai |
| PGD | Thuật toán lặp tạo nhiễu bằng gradient + chiếu về [−ε,ε] |
| ε (epsilon) | Biên độ nhiễu tối đa (chuẩn L-inf) |
| EoT | Tối ưu nhiễu qua nhiều phép biến đổi để nhiễu bền |
| STE | Mẹo cho gradient chảy qua phép làm tròn |
| Perceptual mask | Bản đồ giấu nhiễu vào vùng nhiều chi tiết |
| PSNR / SSIM | Đo độ giống ảnh với mắt người |
| Transferability | Khả năng nhiễu chuyển sang model khác (ở đây: kém) |
