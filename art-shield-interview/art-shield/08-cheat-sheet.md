# 08 — Cheat Sheet (lướt phút chót)

## Pitch 30 giây
> Art Shield bảo vệ tranh họa sĩ khỏi việc bị AI cào đi train. Thêm nhiễu cực nhỏ, vô hình với mắt người, nhưng làm CLIP trích đặc trưng sai. Tự cài PGD trên PyTorch, dùng EoT cho nhiễu bền qua nén/resize, perceptual masking để giấu nhiễu. Có công cụ đo độ bền và đo transfer — và thẳng thắn rằng không tuyệt đối.

## Công thức cốt lõi
- **Mục tiêu**: tối thiểu hóa `cosine(encode(image+δ), encode(image))`, với `‖δ‖∞ ≤ ε`.
- **Cập nhật PGD**: `δ -= step_size · sign(∇δ similarity)` → chiếu về `[−ε,ε]` → giữ ảnh ∈ `[0,1]`.
- **Ngưỡng bảo vệ**: cosine với gốc **< 0.5** (càng thấp càng tốt; âm = rất mạnh).

## Số liệu nhớ sẵn
| | Độ bền | SSIM |
|--|--|--|
| PGD ngây thơ ε=0.03 | 0/9 ❌ | 0.83 |
| EoT ε=0.06 | 9/9 ✅ | 0.60 |
| EoT+tri giác ε=0.04 | 9/9 ✅ | 0.79 |
| Ảnh thật EoT+tri giác ε=0.08 | 9/9 ✅ | **0.94** (cosine −0.42) |

Transfer: nhắm B/32 → trên B/32 cosine **−0.49** (tốt), trên B/16 **+0.84** (vô dụng).

## Thuật ngữ 1 dòng
- **Embedding**: vector 512 chiều mô tả ảnh (CLIP).
- **Cosine**: độ giống hướng 2 vector; 1=giống, <0.5=đã bảo vệ.
- **PGD**: lặp gradient + chiếu để tạo nhiễu.
- **ε**: biên độ nhiễu tối đa (L-inf).
- **EoT**: tối ưu nhiễu qua nhiều phép biến đổi → nhiễu bền.
- **STE**: cho gradient qua phép làm tròn (`x+(q-x).detach()`).
- **Perceptual mask**: giấu nhiễu vào vùng rậm chi tiết (std cục bộ).
- **Transferability**: khả năng nhiễu chuyển sang model khác (ở đây: kém).

## Bản đồ file
| File | Vai trò |
|------|---------|
| `clip_model.py` | Bọc CLIP, encode giữ gradient theo pixel |
| `attack.py` | `pgd_protect`, `pgd_protect_eot`, `..._multi` |
| `diff_transforms.py` | Biến đổi KHẢ VI (train EoT) |
| `transforms.py` | Biến đổi PIL (TEST độ bền, 9 phép) |
| `perceptual.py` | Bản đồ che tri giác |
| `metrics.py` | PSNR, SSIM |
| `pipeline.py` | `protect_image`, `run_robustness`, lazy singleton CLIP |
| `backend/main.py` | FastAPI `/api/protect`, `/api/verify` |
| `backend/config.py` | PRESETS Nhẹ/Vừa/Mạnh, MAX_IMAGE_SIDE=512 |

## Luồng bảo vệ (web)
`POST /api/protect` → `_load_and_fit` (≤512px) → `protect_image` → `perceptual_mask` → `pgd_protect_eot` → đo cosine/PSNR/SSIM → lưu original+protected → JSON.

## Luồng kiểm tra độ bền
`POST /api/verify` (uid) → đọc 2 ảnh → `run_robustness` → áp 9 phép tẩy nhiễu lên ảnh bảo vệ → đếm cosine<0.5 → bảng held/total.

## Preset
| | ε | steps | eot_samples |
|--|--|--|--|
| Nhẹ | 0.04 | 60 | 4 |
| Vừa | 0.06 | 90 | 4 |
| Mạnh | 0.09 | 140 | 5 |

## Câu "tủ" về hạn chế
> "Không công cụ nào miễn nhiễm 100%, kể cả Glaze. Nhiễu nhắm một model không chuyển tốt sang model khác kiến trúc — tôi có thực nghiệm chứng minh. Tôi coi đây là lớp ma sát tăng chi phí cho kẻ lạm dụng, kết hợp watermark và điều khoản sử dụng."

## Bẫy hay gặp
- Mount `/` StaticFiles phải đặt **cuối** (sau /api).
- `F.interpolate` (khả vi) ≠ PIL resize (cắt gradient).
- `round` chặn gradient → cần STE.
- Đo bằng **cosine**, không phải đảo nhãn phân loại.
- EoT chậm vì encode nhiều lần/vòng → giới hạn 512px.
