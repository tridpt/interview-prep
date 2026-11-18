# 07 — Câu hỏi phỏng vấn (kèm trả lời mẫu)

Sắp xếp từ cơ bản → nâng cao. Mỗi câu có gợi ý trả lời ngắn gọn để bạn diễn đạt lại bằng lời mình.

---

## A. Tổng quan & động lực (1–10)

**1. Art Shield làm gì trong một câu?**
Thêm một lớp nhiễu cực nhỏ, vô hình với mắt người, vào tranh để làm mô hình AI (CLIP) trích đặc trưng sai — khiến tranh khó bị dùng để huấn luyện AI.

**2. Vấn đề thực tế nó giải quyết?**
Tranh họa sĩ bị cào hàng loạt để train mô hình vẽ tự động mà không xin phép; Art Shield tăng chi phí/độ khó cho việc đó.

**3. Nó giống công cụ nào ngoài thực tế?**
Glaze và Nightshade của Đại học Chicago — cùng ý tưởng nhiễu đối kháng bảo vệ tác phẩm.

**4. Vì sao nhắm CLIP mà không phải mô hình khác?**
CLIP là bộ mã hóa ảnh phổ biến, là "mắt" của nhiều pipeline AI tạo ảnh/lọc dữ liệu. Làm hỏng đặc trưng CLIP làm tranh khó dùng cho các pipeline đó.

**5. Công nghệ chính?**
PyTorch (gradient), CLIP qua transformers, PGD/EoT tự cài, PSNR/SSIM (scikit-image), FastAPI + JS thuần cho web.

**6. Dự án có mấy giai đoạn?**
3: (1) chứng minh kỹ thuật, (2) đo + nâng cấp độ bền (EoT, masking), (3) web app.

**7. Đầu vào/đầu ra của hệ thống?**
Vào: một ảnh + mức bảo vệ. Ra: ảnh đã thêm nhiễu + các chỉ số (cosine với gốc, PSNR, SSIM) + (tùy chọn) báo cáo độ bền.

**8. Ai là người dùng?**
Họa sĩ/người sáng tạo muốn bảo vệ tác phẩm trước khi đăng lên mạng.

**9. Vì sao có cả CLI lẫn web?**
CLI cho người kỹ thuật tinh chỉnh tham số/chạy thực nghiệm; web cho họa sĩ không rành kỹ thuật (kéo-thả, preset).

**10. Hạn chế lớn nhất?**
Không bảo vệ tuyệt đối; nhiễu nhắm một model **không chuyển** tốt sang model khác kiến trúc; có thể yếu đi khi ảnh bị xử lý mạnh.

---

## B. Khái niệm nền tảng (11–28)

**11. Embedding là gì?**
Vector số mô tả nội dung ảnh (CLIP: 512 chiều). Ảnh giống nhau → embedding gần nhau.

**12. Cosine similarity đo gì? Vì sao dùng nó?**
Cosin góc giữa hai vector. Vì embedding đã chuẩn hóa L2, cosine = dot product, đo "AI thấy hai ảnh giống nhau bao nhiêu". Dùng làm thước đo bảo vệ chính.

**13. Vì sao ngưỡng bảo vệ là cosine < 0.5?**
Dưới 0.5 nghĩa là embedding ảnh bảo vệ đã lệch đủ xa ảnh gốc → AI "thấy" thứ khác hẳn → coi như bảo vệ thành công.

**14. Nhiễu đối kháng là gì? Khác nhiễu ngẫu nhiên thế nào?**
Là nhiễu nhỏ **có chủ đích** (tìm bằng gradient) làm model dự đoán sai; nhiễu ngẫu nhiên thường vô hại vì không nhắm hướng nhạy cảm của model.

**15. Vì sao thêm nhiễu nhỏ lại làm AI sai nhiều?**
Vì mạng nơ-ron có những "hướng nhạy cảm" trong không gian nhiều chiều; dịch pixel đúng hướng đó làm output đổi mạnh dù mắt người không thấy.

**16. PGD là gì? Mô tả vòng lặp.**
Projected Gradient Descent: lặp nhiều bước, mỗi bước đi theo dấu gradient để giảm similarity, rồi chiếu nhiễu về `[−ε,ε]` và giữ ảnh trong `[0,1]`.

**17. ε (epsilon) là gì? Đánh đổi?**
Biên độ nhiễu tối đa mỗi pixel (chuẩn L-inf). Lớn = bảo vệ mạnh nhưng dễ lộ; nhỏ = kín nhưng yếu.

**18. Vì sao dùng `sign(gradient)` thay vì gradient đầy đủ?**
PGD/FGSM-style: dùng dấu cho bước đi đều, ổn định, không bị ảnh hưởng bởi độ lớn gradient dao động.

**19. "Chiếu" (project) trong PGD nghĩa là gì?**
Sau mỗi bước, cắt nhiễu về lại miền hợp lệ `[−ε,ε]` và đảm bảo ảnh+nhiễu ∈ `[0,1]`. Đảm bảo ràng buộc không bị vi phạm.

**20. Vì sao nhiễu PGD ngây thơ bị tẩy dễ?**
Nó nằm ở tần số cao; nén/resize/làm mờ là bộ lọc làm mượt, xóa tần số cao → nhiễu biến mất.

**21. EoT là gì?**
Expectation over Transformation: tối ưu nhiễu qua nhiều phép biến đổi ngẫu nhiên rồi lấy trung bình, ép nhiễu sống sót qua các phép xử lý ảnh.

**22. Vì sao biến đổi trong EoT phải khả vi?**
Để gradient chảy qua được phép biến đổi mà tối ưu nhiễu; nếu không khả vi thì không có gradient để học.

**23. STE là gì? Dùng ở đâu?**
Straight-through estimator: cho gradient đi qua phép làm tròn (forward làm tròn, backward đồng nhất). Dùng trong `ste_quantize` để mô phỏng nén.

**24. Perceptual masking là gì?**
Giấu nhiễu vào vùng nhiều chi tiết (mắt khó thấy), ghìm nhiễu ở vùng phẳng. Dựa trên độ lệch chuẩn cục bộ của độ sáng.

**25. PSNR và SSIM khác nhau?**
PSNR đo sai khác pixel (dB, càng cao càng giống); SSIM mô phỏng cảm nhận cấu trúc của mắt người (0..1, càng gần 1 càng giống). SSIM gần với cảm nhận người hơn.

**26. CLIP phân loại zero-shot kiểu gì?**
Tính cosine giữa embedding ảnh và embedding các mô tả văn bản, softmax → nhãn cosine cao nhất.

**27. Transferability là gì? Kết quả trong dự án?**
Khả năng nhiễu chuyển sang model khác. Thử nghiệm chỉ so sánh CLIP B/32 với B/16 và thấy transfer kém; **chưa thử GPT-4o**.

**28. Threat model của bài toán?**
Phòng thủ = họa sĩ (tạo nhiễu); tấn công = người cào dữ liệu (tẩy nhiễu). Thắng nếu nhiễu sống sót.

---

## C. Triển khai & code (29–48)

**29. Vì sao đóng băng tham số CLIP?**
Ta không train model, chỉ tối ưu nhiễu. Đóng băng tiết kiệm bộ nhớ và đảm bảo gradient chỉ theo ảnh.

**30. Vì sao `encode_image` nhận tensor [0,1] và tự chuẩn hóa?**
Để gradient chảy về tận pixel gốc — điều kiện bắt buộc để tối ưu nhiễu. Chuẩn hóa nằm trong đồ thị tính toán.

**31. Vì sao dùng `F.interpolate` resize chứ không PIL?**
`F.interpolate` khả vi nên gradient qua được; PIL resize cắt gradient.

**32. `target_emb` được `.detach()` để làm gì?**
Nó là mốc cố định (embedding ảnh gốc), không cần gradient, detach để autograd không truy ngược.

**33. Trong PGD vì sao `delta -= step_size * grad.sign()` (dấu trừ)?**
Vì ta muốn **giảm** similarity (đẩy xa), nên đi ngược hướng gradient của similarity.

**34. Ba phép "chiếu" cuối mỗi vòng làm gì?**
(1) cắt về `[−ε,ε]`; (2) nếu có mask, cắt theo `ε*mask`; (3) chỉnh delta để `image+delta ∈ [0,1]`.

**35. `sample_transform` hoạt động ra sao?**
Bật/tắt ngẫu nhiên 4 phép (resize, blur, quantize, noise) với tham số ngẫu nhiên — tạo một mẫu EoT đa dạng.

**36. Vì sao EoT cộng thêm trường hợp ảnh "sạch"?**
Để nhiễu vẫn đánh lừa được ảnh nguyên bản chưa qua xử lý, không chỉ ảnh đã biến đổi.

**37. `pgd_protect_eot_multi` khác gì? Mục đích?**
Tối ưu nhiễu trên nhiều CLIP cùng lúc → khai thác đặc trưng chung → tăng khả năng chuyển sang model lạ.

**38. `perceptual_mask` tính "độ chi tiết" thế nào?**
Độ lệch chuẩn cục bộ của độ sáng qua cửa sổ trượt (avg_pool của x² trừ bình phương avg_pool của x).

**39. Vì sao chuẩn hóa mask theo phân vị 0.9 thay vì max?**
Để bền với vài pixel cực trị (outlier) làm lệch thang đo.

**40. `to_uint8` và vai trò khi đo metrics?**
Chuyển [0,1] float → uint8 [0,255] để PSNR/SSIM tính trên cùng thang với ảnh thật đã lưu.

**41. Vì sao tách `diff_transforms.py` và `transforms.py`?**
Train cần khả vi (torch) để có gradient; test cần phép thật (PIL/JPEG) để khách quan.

**42. `pipeline.get_clip()` dùng mẫu gì? Lợi ích?**
Lazy singleton: nạp CLIP một lần, tái dùng cho mọi request → request đầu chậm, sau nhanh.

**43. `protect_image` trả về gì?**
`(ảnh PIL đã bảo vệ, dict metrics)` gồm cosine, protected (bool), psnr, ssim, nhãn trước/sau.

**44. `run_robustness` trả về gì?**
`{baseline_cosine, threshold, rows[], held, total}` — bảng từng phép tẩy nhiễu và số phép sống sót.

**45. Vì sao mount `/` cuối cùng trong FastAPI?**
Để StaticFiles `/` không che các route `/api/*` và `/files`.

**46. Vì sao chèn `BASE_DIR` vào `sys.path` trong backend?**
Để import gói `shield/` ở thư mục gốc từ trong `backend/`.

**47. Preset trong config gồm gì? Vì sao preset hóa?**
`epsilon, steps, mask_min, eot_samples` cho Nhẹ/Vừa/Mạnh. Preset hóa để người dùng cuối không phải hiểu tham số kỹ thuật.

**48. Vì sao thêm `?t=Date.now()` vào src ảnh ở frontend?**
Chống cache trình duyệt, đảm bảo hiển thị ảnh mới nhất sau mỗi lần xử lý.

---

## D. Câu hỏi sâu / phản biện (49–62)

**49. Làm sao biết nhiễu "vô hình"? Đo bằng gì?**
PSNR/SSIM; ví dụ ảnh thật đạt SSIM 0.94 nghĩa là khác biệt cấu trúc rất nhỏ.

**50. Nếu kẻ tấn công dùng model bạn không biết thì sao?**
Nhiễu có thể vô dụng (transfer kém). Ensemble giúp phần nào nhưng không phủ hết — đây là giới hạn cốt lõi, tôi nói thẳng điều đó.

**51. Vì sao không dùng thư viện adversarial có sẵn?**
Tự cài để hiểu sâu cơ chế và kiểm soát hoàn toàn (mask, EoT tùy biến). Mục tiêu dự án là học tập/nghiên cứu.

**52. Đánh đổi ε lớn?**
Bảo vệ mạnh hơn nhưng nhiễu dễ thấy (SSIM/PSNR giảm). Preset "Mạnh" chấp nhận lộ hơn để bền hơn.

**53. Vì sao đo bằng cosine chứ không bằng "AI đoán sai nhãn"?**
Nhãn có thể không đổi dù embedding lệch xa; cosine phản ánh đúng mức "AI thấy khác" hơn.

**54. EoT có làm chậm không? Vì sao?**
Có — mỗi vòng encode ảnh `eot_samples+1` lần qua CLIP. Đó là lý do giới hạn 512px và preset điều chỉnh `eot_samples`.

**55. Nếu chạy trên GPU thì khác gì?**
Nhanh hơn nhiều lần (CLIP chạy trên CUDA); có thể tăng steps/eot_samples và xử lý ảnh lớn hơn.

**56. Rủi ro đạo đức của công cụ?**
Nó là phòng thủ (bảo vệ tác phẩm), nhưng cùng kỹ thuật adversarial có thể dùng tấn công. Dự án định vị rõ là bảo vệ + học tập.

**57. Làm sao mở rộng sang model đóng (GPT-4o)?**
Không có gradient của model đóng → không thể tối ưu trực tiếp. Chỉ trông vào transfer (kém). Đây là rào cản căn bản của black-box.

**58. Cải tiến tiếp theo bạn nghĩ tới?**
Ensemble nhiều họ model hơn, tối ưu nhiễu theo không gian tần số, hoặc kết hợp watermark; thêm auth nếu deploy thật.

**59. Vì sao SSIM trả về None đôi khi?**
Khi thiếu scikit-image — fallback mềm để app vẫn chạy, chỉ là không có chỉ số SSIM.

**60. Bài học lớn nhất từ dự án?**
Nhiễu đối kháng "đẹp trên giấy" nhưng vô dụng nếu không bền; EoT là bắt buộc; và phải trung thực rằng không có lá chắn tuyệt đối.

**61. Nếu phải bảo vệ 10.000 ảnh thì sao?**
Cần GPU và xử lý theo lô (batch); CPU + web hiện tại chỉ hợp lẻ tẻ. Có thể đưa lên Colab/GPU server.

**62. Cosine âm (vd −0.42) nghĩa là gì?**
Embedding ảnh bảo vệ gần như **ngược hướng** ảnh gốc — AI thấy nội dung trái ngược, mức bảo vệ rất mạnh.

---

## Mẹo trả lời
- Luôn nối **lý thuyết với code thật** ("tôi dùng `.detach()` ở `target_emb` vì...").
- Khi bị hỏi về giới hạn, **thừa nhận thẳng** rồi nêu hướng cải thiện — đừng phòng thủ.
- Có sẵn vài **con số** (0/9 → 9/9, SSIM 0.94, cosine −0.49 vs +0.84) để tăng sức thuyết phục.
