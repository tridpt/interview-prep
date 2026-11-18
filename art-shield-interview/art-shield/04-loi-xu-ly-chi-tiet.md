# 04 — Chi tiết lõi xử lý (đọc từng hàm)

File này đi qua **từng hàm quan trọng** trong `shield/`. Đây là phần "ruột" — hiểu được phần này là bạn nắm chắc dự án.

---

## A. `clip_model.py` — Bọc CLIP

### Mục tiêu thiết kế
Quan trọng nhất: `encode_image` nhận ảnh ở dạng **tensor pixel trong [0,1]** và **tự chuẩn hóa bên trong**. Nhờ vậy **gradient chảy ngược được về từng pixel** — điều kiện *bắt buộc* để tạo nhiễu đối kháng.

### `__init__`
```python
self.model = CLIPModel.from_pretrained(model_name).to(device).eval()
for p in self.model.parameters():
    p.requires_grad_(False)   # KHÔNG học trọng số model, chỉ tối ưu nhiễu
```
- `.eval()`: tắt dropout/batchnorm cập nhật.
- **Đóng băng tham số**: ta không train CLIP; ta chỉ tối ưu `delta` (nhiễu). Đóng băng giúp tiết kiệm bộ nhớ và đảm bảo gradient chỉ tính theo ảnh đầu vào.
- Lưu `mean`/`std` của CLIP dạng tensor `(1,3,1,1)` để broadcast khi chuẩn hóa.

### `preprocess_pixels`
```python
pixels = F.interpolate(pixels, size=(224,224), mode="bicubic")  # resize về 224
pixels = pixels.clamp(0,1)
return (pixels - self._mean) / self._std                        # chuẩn hóa
```
- CLIP cần đầu vào 224×224, đã chuẩn hóa theo mean/std của OpenAI.
- Dùng `F.interpolate` (khả vi) chứ **không** dùng resize của PIL → gradient vẫn chảy qua bước resize.

### `_coerce` — tương thích nhiều phiên bản transformers
`get_image_features` ở bản cũ trả thẳng tensor, bản mới (transformers 5.x) trả về object có `image_embeds`/`pooler_output`. Hàm này dò các thuộc tính để luôn lấy được vector embedding. → Một chi tiết "phòng thủ" cho thấy tác giả quan tâm tính tương thích.

### `encode_image`
```python
normalized = self.preprocess_pixels(pixels)
out = self.model.get_image_features(pixel_values=normalized)
return F.normalize(self._coerce(out), dim=-1)   # chuẩn hóa L2 (độ dài = 1)
```
Chuẩn hóa L2 để cosine similarity = phép nhân chấm đơn giản.

### `encode_text` và `classify`
- `encode_text`: token hóa danh sách mô tả → embedding text (có `@torch.no_grad()` vì không cần gradient cho text).
- `classify`: tính `logits = (img @ txt.T) * logit_scale.exp()`, rồi `softmax` → xác suất từng nhãn, sắp giảm dần. Đây là phân loại zero-shot của CLIP. **Lưu ý**: trong dự án, nhãn phân loại chỉ là *thông tin phụ*; thước đo chính là cosine.

---

## B. `attack.py` — Trái tim của dự án

### `pgd_protect` — PGD cơ bản
```python
with torch.no_grad():
    target_emb = clip.encode_image(image).detach()  # mốc cần né xa

delta = torch.empty_like(image).uniform_(-epsilon, epsilon)  # nhiễu ngẫu nhiên
delta.requires_grad_(True)

for i in range(steps):
    adv = (image + delta).clamp(0, 1)
    adv_emb = clip.encode_image(adv)
    similarity = (adv_emb * target_emb).sum(dim=-1).mean()   # cosine với gốc

    similarity.backward()                  # gradient của similarity theo delta
    with torch.no_grad():
        delta -= step_size * delta.grad.sign()   # ĐI NGƯỢC để GIẢM similarity
        delta.clamp_(-epsilon, epsilon)          # chiếu về [−ε, ε]
        if mask is not None:                     # giấu nhiễu theo mask
            bound = epsilon * mask
            delta.copy_(torch.clamp(delta, -bound, bound))
        delta.copy_((image + delta).clamp(0,1) - image)  # giữ ảnh trong [0,1]
```

Điểm cần giải thích khi phỏng vấn:
- **`target_emb` được `.detach()`**: nó là hằng số (mốc), không cần gradient.
- **`delta -= step_size * sign(grad)`**: ta muốn **giảm** similarity (đẩy embedding ra xa), nên đi *ngược* hướng gradient của similarity. Dùng `sign()` (PGD/FGSM-style) cho bước đi ổn định.
- **3 phép chiếu liên tiếp**: (1) cắt vào `[−ε,ε]`; (2) nếu có mask thì cắt thêm theo `ε*mask`; (3) đảm bảo `image+delta ∈ [0,1]` bằng cách điều chỉnh lại `delta`. Thứ tự này đảm bảo mọi ràng buộc đều được tôn trọng cuối mỗi vòng.
- `step_size` mặc định `2.5 * epsilon / steps` — quy tắc kinh nghiệm phổ biến trong PGD.

### `pgd_protect_eot` — PGD bền vững (điểm nhấn lớn nhất)
Khác `pgd_protect` ở **cách tính similarity** mỗi vòng:
```python
sims = []
for _ in range(eot_samples):
    transformed = sample_transform(adv)        # biến đổi NGẪU NHIÊN (khả vi)
    emb = clip.encode_image(transformed)
    sims.append((emb * target_emb).sum(-1).mean())
emb_clean = clip.encode_image(adv)             # thêm cả trường hợp KHÔNG biến đổi
sims.append((emb_clean * target_emb).sum(-1).mean())
similarity = torch.stack(sims).mean()          # KỲ VỌNG (trung bình)
```
- Trung bình similarity qua `eot_samples` phép biến đổi ngẫu nhiên + 1 lần ảnh sạch.
- Tối ưu trên trung bình này → nhiễu phải sống sót qua *mọi* phép → bền.
- Vì sao thêm ảnh sạch (`emb_clean`)? Để nhiễu vẫn đánh lừa được ảnh **nguyên bản** chưa qua xử lý, không chỉ ảnh đã biến đổi.

### `pgd_protect_eot_multi` — Ensemble nhiều model
Giống EoT nhưng vòng lặp duyệt **nhiều CLIP** (`clips`), cộng dồn similarity trên từng model:
```python
for c, tgt in zip(clips, targets):
    for _ in range(eot_samples):
        emb = c.encode_image(sample_transform(adv))
        sims.append((emb * tgt).sum(-1).mean())
    sims.append((c.encode_image(adv) * tgt).sum(-1).mean())
```
- Mục đích: nhiễu phải làm hỏng đặc trưng của **mọi** model trong tập → khai thác đặc trưng *chung* → tăng khả năng chuyển sang model lạ.
- Dùng trong `eval_transfer.py` để so cách (A) chỉ nhắm B/32 vs (B) ensemble B/32+B/16.

---

## C. `diff_transforms.py` — Biến đổi KHẢ VI (cho EoT)

Tất cả bằng torch để gradient chảy qua.

| Hàm | Làm gì | Mô phỏng |
|-----|--------|----------|
| `diff_gaussian_blur` | Conv với kernel Gaussian tự sinh từ sigma | Làm mờ / khử nhiễu |
| `diff_resize_roundtrip` | Thu nhỏ rồi phóng lại bằng `F.interpolate` | Resize của kẻ thu thập |
| `ste_quantize` | Lượng tử hóa màu bằng **STE** | Nén / giảm bit màu |
| `add_noise` | Cộng nhiễu Gaussian | Khử nhiễu/nén làm méo pixel |

### `sample_transform` — bộ tạo mẫu EoT
```python
if random.random() < 0.8: out = diff_resize_roundtrip(out, scale=uniform(0.4, 0.85))
if random.random() < 0.8: out = diff_gaussian_blur(out, sigma=uniform(0.5, 1.5))
if random.random() < 0.6: out = ste_quantize(out, bits=choice([4,5,6]))
if random.random() < 0.5: out = add_noise(out, std=uniform(0.005, 0.02))
return out.clamp(0, 1)
```
Mỗi phép **bật/tắt ngẫu nhiên** với tham số ngẫu nhiên → mô phỏng việc "ta không biết trước kẻ thu thập sẽ xử lý ảnh kiểu gì". Đa dạng hóa này chính là tinh thần của EoT.

### `ste_quantize` — giải thích kỹ
```python
levels = 2**bits - 1
q = torch.round(x.clamp(0,1) * levels) / levels
return x + (q - x).detach()
```
Forward cho ra `q` (đã làm tròn); backward: vì `(q-x)` bị `.detach()`, đạo hàm `d/dx = 1`. Đây là cách "lừa" autograd để gradient đi qua phép làm tròn vốn không khả vi.

---

## D. `transforms.py` — Biến đổi để TEST (PIL, không khả vi)

Mô phỏng đúng những gì kẻ thu thập làm trong thực tế:
- `jpeg_compress`: lưu JPEG ở quality cho trước rồi mở lại (nén **thật**, không phải xấp xỉ).
- `resize_roundtrip`: thu nhỏ/phóng lại bằng PIL BICUBIC.
- `gaussian_blur`, `median_filter`, `bit_quantize`.
- `build_default_suite()`: trả về 9 phép gồm JPEG 90/70/50, resize 50%/75%, blur r=1.0, median 3×3, 4-bit màu, và 1 phép **kết hợp** khắc nghiệt (resize 50% + JPEG 70).

> Cặp đôi `diff_transforms` (train) ↔ `transforms` (test) là một thiết kế đẹp: train bằng phiên bản khả vi *gần đúng*, nhưng test bằng phiên bản *thật* để kết quả khách quan.

---

## E. `perceptual.py` — Bản đồ che tri giác

```python
lum = 0.299*r + 0.587*g + 0.114*b          # độ sáng cảm nhận
mean = avg_pool2d(lum_padded, window)       # trung bình cục bộ
mean_sq = avg_pool2d(lum_padded**2, window)
std = sqrt(mean_sq - mean**2)               # ĐỘ LỆCH CHUẨN cục bộ = độ "rậm chi tiết"
ref = quantile(std, 0.9)                     # mốc chuẩn hóa (tránh điểm cực trị)
detail = (std / ref).clamp(0, 1)
mask = m_min + (1 - m_min) * detail          # co về [m_min, 1]
```
- **std cục bộ cao** = vùng nhiều cạnh/chi tiết = cho nhiễu mạnh.
- **std thấp** = vùng phẳng = ghìm nhiễu xuống `m_min` (giữ vùng trơn sạch).
- Chuẩn hóa theo **phân vị 0.9** thay vì max → bền với vài pixel cực trị.
- Trả mask `(1,1,H,W)` để nhân với biên độ nhiễu trong `attack.py`.

---

## F. `metrics.py` — Đo chất lượng

- `to_uint8`: chuyển `[0,1]` float → `uint8 [0,255]`.
- `psnr`: `20*log10(255/sqrt(MSE))`. MSE=0 → trả `inf` (ảnh y hệt).
- `ssim`: gọi `skimage.metrics.structural_similarity`; nếu thiếu scikit-image thì trả `None` (fallback mềm, app vẫn chạy).

---

## G. `pipeline.py` — Keo dán mọi thứ

- `get_clip()`: lazy singleton (mục 6, file 03).
- `protect_image(img, epsilon, steps, use_eot, use_perceptual, mask_min, eot_samples)`: làm trọn quy trình bảo vệ và trả `(ảnh PIL, dict metrics)`. Đây là hàm web gọi.
- `run_robustness(original, protected, threshold)`: chạy bộ test, trả `{baseline_cosine, rows, held, total}`.
- Hằng `PROTECT_THRESHOLD = 0.5` và `LABELS` đặt tập trung ở đây.

> Khi phỏng vấn, nếu được hỏi "luồng từ ảnh vào đến kết quả ra", hãy kể theo `protect_image`: pil_to_tensor → perceptual_mask → pgd_protect_eot → đo cosine/PSNR/SSIM → tensor_to_pil.
