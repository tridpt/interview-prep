# 03 — Kiến trúc hệ thống

## 1. Bức tranh tổng thể

Art Shield có **2 mặt sử dụng** chung một lõi:

```
                    ┌─────────────────────────────┐
   CLI (dòng lệnh)  │   LÕI dùng chung: shield/    │   Web app
   protect.py ─────►│                             │◄──── backend/main.py
   robustness.py ──►│  clip_model.py  attack.py   │      (FastAPI)
   eval_transfer ──►│  perceptual.py  metrics.py  │            ▲
                    │  transforms.py  diff_trans  │            │
                    │  pipeline.py (keo lai)      │      frontend/ (HTML/JS)
                    └─────────────────────────────┘
```

- **CLI**: dành cho người kỹ thuật, chạy thử nghiệm, tinh chỉnh tham số (`--epsilon`, `--steps`, `--eot`...).
- **Web app**: dành cho họa sĩ, kéo-thả ảnh, chọn preset.
- Cả hai gọi vào `shield/` — đặc biệt `pipeline.py` là điểm "keo" gom logic dùng chung cho web.

## 2. Các lớp (layers) và trách nhiệm

| Lớp | File | Trách nhiệm |
|-----|------|-------------|
| **Mô hình** | `shield/clip_model.py` | Bọc CLIP: encode ảnh/text (giữ gradient theo pixel), phân loại |
| **Thuật toán** | `shield/attack.py` | PGD, PGD-EoT, ensemble — tạo nhiễu |
| **Biến đổi (train)** | `shield/diff_transforms.py` | Mờ/resize/nén KHẢ VI cho EoT |
| **Biến đổi (test)** | `shield/transforms.py` | Mờ/resize/nén bằng PIL để đo độ bền |
| **Tri giác** | `shield/perceptual.py` | Bản đồ che giấu nhiễu |
| **Đo lường** | `shield/metrics.py` | PSNR, SSIM |
| **Điều phối** | `shield/pipeline.py` | Gom mọi thứ thành `protect_image()` + `run_robustness()` |
| **Giao tiếp web** | `backend/main.py` | FastAPI: `/api/protect`, `/api/verify`, phục vụ file tĩnh |
| **Cấu hình** | `backend/config.py` | Preset Nhẹ/Vừa/Mạnh, đường dẫn, giới hạn |
| **Giao diện** | `frontend/` | Kéo-thả, hiển thị kết quả, gọi API |

> Nguyên tắc thiết kế chủ đạo: **tách phần "thuật toán thuần" (shield/) khỏi phần "giao tiếp" (CLI / web)**. Nhờ vậy đổi giao diện không đụng thuật toán, và ngược lại.

## 3. Luồng dữ liệu — chức năng "Bảo vệ ảnh"

Đây là luồng quan trọng nhất. Ví dụ qua web:

```
1. Người dùng kéo ảnh + chọn preset "vừa" → frontend gửi POST /api/protect
2. backend/main.py:
   - _load_and_fit(): đọc ảnh, thu nhỏ nếu cạnh dài > MAX_IMAGE_SIDE (512)
   - tra PRESETS["vua"] → {epsilon, steps, mask_min, eot_samples}
   - gọi pipeline.protect_image(...)
3. pipeline.protect_image():
   - get_clip(): nạp CLIP (lazy singleton — chỉ nạp 1 lần)
   - pil_to_tensor(): ảnh PIL → tensor (1,3,H,W) trong [0,1]
   - perceptual_mask(): tạo bản đồ che
   - pgd_protect_eot(): VÒNG LẶP TỐI ƯU NHIỄU (phần nặng nhất, ~1-2 phút CPU)
   - đo cosine với gốc, PSNR, SSIM → dict metrics
4. backend lưu {uid}_original.png + {uid}_protected.png vào storage/web/
5. trả JSON: {uid, metrics, original_url, protected_url}
6. frontend hiển thị ảnh trước/sau + chỉ số
```

## 4. Luồng dữ liệu — chức năng "Kiểm tra độ bền"

```
1. Người dùng bấm "Kiểm tra độ bền" → POST /api/verify với uid
2. backend đọc lại 2 ảnh đã lưu (original, protected)
3. pipeline.run_robustness():
   - tính embedding ảnh gốc làm mốc
   - với từng phép trong build_default_suite() (JPEG 90/70/50, resize, blur, median, quantize, kết hợp):
       • áp phép đó lên ảnh ĐÃ BẢO VỆ
       • tính cosine với ảnh gốc
       • "còn bảo vệ" nếu cosine < ngưỡng (0.5)
   - đếm số phép sống sót (held/total)
4. trả bảng kết quả → frontend render thành bảng
```

## 5. Vì sao tách `diff_transforms.py` và `transforms.py`?

Đây là một quyết định kiến trúc tinh tế, hay được hỏi:

| | `diff_transforms.py` | `transforms.py` |
|--|---------------------|-----------------|
| Công cụ | torch (tensor) | PIL (ảnh) |
| Khả vi? | **Có** (gradient chảy qua) | Không |
| Dùng khi | **TRAIN** nhiễu (EoT) | **TEST** độ bền |
| Vì sao | EoT cần gradient để tối ưu nhiễu qua phép biến đổi | Test chỉ cần mô phỏng đúng thực tế (JPEG thật, resize thật) |

Nếu dùng chung một loại sẽ sai: dùng PIL khi train → không có gradient; dùng torch khi test → không phản ánh đúng JPEG thật mà kẻ tấn công dùng.

## 6. Lazy singleton — nạp CLIP một lần

CLIP nặng (~350MB) và mất vài giây để nạp. `pipeline.py` dùng mẫu **lazy singleton**:
```python
_clip = None
def get_clip():
    global _clip
    if _clip is None:
        _clip = ClipWrapper()   # chỉ nạp lần đầu
    return _clip
```
Nhờ vậy: request đầu chậm (nạp model), các request sau nhanh (tái dùng). Hợp với web server chạy lâu dài.

## 7. Cách backend phục vụ frontend

`backend/main.py` mount 2 thư mục tĩnh, **đúng thứ tự**:
```python
app.mount("/files", StaticFiles(directory=WEB_STORAGE_DIR))   # ảnh đã xử lý
app.mount("/", StaticFiles(directory=FRONTEND_DIR, html=True)) # giao diện
```
> ⚠️ Mount `/` phải đặt **cuối cùng**, sau các route `/api/*` và `/files`. Nếu đặt trước, nó sẽ "nuốt" mọi request và che mất các API. Đây là một bẫy hay gặp với FastAPI StaticFiles.

## 8. Giới hạn kích thước để CPU chạy kịp

Vòng lặp PGD-EoT rất nặng (mỗi vòng encode ảnh nhiều lần qua CLIP). Để web dùng được trên CPU:
- `MAX_IMAGE_SIDE = 512`: ảnh lớn bị thu về tối đa 512px.
- Preset điều chỉnh `steps` và `eot_samples` để cân bằng tốc độ/chất lượng (Nhẹ ~1 phút, Mạnh ~3 phút).

## 9. Sơ đồ phụ thuộc module

```
protect.py ───────┐
robustness.py ────┤
eval_transfer.py ─┼──► shield/attack.py ──► shield/clip_model.py ──► transformers (CLIP)
                  │         │                                   └──► torch
backend/main.py ──┘         └──► shield/diff_transforms.py ──► torch
       │
       └──► shield/pipeline.py ──► attack + clip_model + perceptual + metrics + transforms
              │
backend/config.py (PRESETS, paths)
```

`pipeline.py` là trung tâm: nó import gần như mọi module trong `shield/` và là thứ duy nhất mà `backend` cần biết.
