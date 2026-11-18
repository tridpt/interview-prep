# 05 — Chi tiết Web app (Giai đoạn 3)

Web app là phần "đóng gói" lõi kỹ thuật thành công cụ cho họa sĩ không rành kỹ thuật. Gồm **backend FastAPI** và **frontend HTML/CSS/JS thuần**.

---

## A. Backend — `backend/main.py`

### Khởi tạo và CORS
```python
app = FastAPI(title="Art Shield", version="0.1.0")
app.add_middleware(CORSMiddleware, allow_origins=["*"], ...)
```
- **CORS** mở `*` để dễ phát triển/demo. (Khi phỏng vấn nên nói thẳng: production cần siết origin lại.)

### Import gói `shield` từ thư mục cha
```python
BASE_DIR = Path(__file__).resolve().parent.parent
if str(BASE_DIR) not in sys.path:
    sys.path.insert(0, str(BASE_DIR))
from shield.pipeline import protect_image, run_robustness
```
- backend nằm trong `backend/`, còn gói `shield/` ở thư mục gốc. Chèn `BASE_DIR` vào `sys.path` để import được. → Một mẹo cấu trúc dự án cần biết.

### `_load_and_fit(upload)`
Đọc ảnh upload bằng PIL, convert RGB; nếu cạnh dài > `MAX_IMAGE_SIDE` (512) thì thu nhỏ giữ tỉ lệ bằng `Image.LANCZOS`. Bắt lỗi đọc file → trả HTTP 400 với thông báo tiếng Việt rõ ràng.

### `GET /api/health`
Trả `{status, device (cpu/cuda), presets}`. Hữu ích để kiểm tra server sống và đang chạy trên CPU hay GPU.

### `POST /api/protect`
```python
async def protect(image: UploadFile, preset: str = Form("vua")):
    if preset not in PRESETS: raise HTTPException(400, ...)
    original = _load_and_fit(image)
    params = PRESETS[preset]
    protected, metrics = protect_image(original, **params, use_eot=True, use_perceptual=True)
    uid = uuid.uuid4().hex[:12]
    original.save(WEB_STORAGE_DIR / f"{uid}_original.png")
    protected.save(WEB_STORAGE_DIR / f"{uid}_protected.png")
    return JSONResponse({uid, preset, metrics, original_url, protected_url})
```
- Nhận ảnh + tên preset, áp tham số preset, gọi lõi.
- Sinh `uid` ngẫu nhiên, lưu **cả ảnh gốc (đã fit) lẫn ảnh bảo vệ** — cần ảnh gốc để bước verify đo cosine.
- Trả URL ảnh để frontend hiển thị.

### `POST /api/verify`
Nhận `uid`, đọc lại 2 ảnh đã lưu, gọi `run_robustness()`, trả bảng kết quả. Nếu không tìm thấy phiên → HTTP 404.

### Phục vụ tĩnh (thứ tự quan trọng)
```python
app.mount("/files", StaticFiles(directory=WEB_STORAGE_DIR))     # ảnh phiên làm việc
app.mount("/", StaticFiles(directory=FRONTEND_DIR, html=True))  # giao diện — ĐẶT CUỐI
```
Mount `/` cuối cùng để không che các route `/api/*`.

---

## B. Cấu hình — `backend/config.py`

```python
MAX_IMAGE_SIDE = 512
PRESETS = {
    "nhe":  {"epsilon": 0.04, "steps": 60,  "mask_min": 0.12, "eot_samples": 4},
    "vua":  {"epsilon": 0.06, "steps": 90,  "mask_min": 0.12, "eot_samples": 4},
    "manh": {"epsilon": 0.09, "steps": 140, "mask_min": 0.12, "eot_samples": 5},
}
```
- **Preset hóa** các tham số kỹ thuật thành 3 mức dễ hiểu cho người dùng cuối.
- `epsilon` tăng dần (nhiễu mạnh hơn = bảo vệ chắc hơn nhưng dễ lộ hơn); `steps`/`eot_samples` cũng tăng → mạnh hơn nhưng chậm hơn.
- `WEB_STORAGE_DIR` được `mkdir(parents=True, exist_ok=True)` ngay khi import → tự tạo thư mục lưu nếu chưa có.

---

## C. Frontend — `frontend/`

Cấu trúc 3 file thuần, không framework:

### `index.html` — 4 phần (card)
1. **Chọn ảnh**: vùng kéo-thả (`#dropzone`) + input file ẩn.
2. **Mức bảo vệ**: 3 radio Nhẹ/Vừa/Mạnh (Vừa mặc định `checked`).
3. **Kết quả** (ẩn ban đầu): so sánh ảnh gốc/bảo vệ, các chỉ số, nút tải + nút "Kiểm tra độ bền".
4. **Lưu ý**: nhắc nhở bảo vệ không tuyệt đối.
Có `#overlay` (spinner) hiển thị khi đang xử lý.

### `app.js` — logic
- **Kéo-thả**: lắng nghe `dragover`/`drop`/`change`, đọc file bằng `FileReader` để xem trước ngay trong dropzone.
- **Bảo vệ** (`protectBtn`):
  ```js
  const form = new FormData();
  form.append("image", selectedFile);
  form.append("preset", preset);
  const res = await fetch("/api/protect", { method: "POST", body: form });
  ```
  Hiển thị overlay khi chờ (1–2 phút), nhận JSON, gọi `renderResult`.
- **`renderResult`**: gắn `src` ảnh (kèm `?t=Date.now()` để **chống cache** trình duyệt), tô màu cosine theo `m.protected` (xanh nếu đạt, đỏ nếu chưa), hiển thị SSIM/PSNR.
- **Kiểm tra độ bền** (`verifyBtn`): gửi `uid`, nhận bảng, `renderVerify` dựng `<table>` với tag "🛡️ còn bảo vệ" / "❌ bị phá".
- **Overlay**: `showOverlay/hideOverlay` bật/tắt lớp phủ + đổi text.

### `style.css`
Theme tối (dark), bố cục card, lưới so sánh ảnh 2 cột, các "metric box", responsive (xuống 1 cột khi màn hình hẹp), animation spinner. Không phụ thuộc thư viện CSS nào.

---

## D. Trải nghiệm người dùng đầu–cuối

```
Kéo ảnh vào ─► chọn "Vừa" ─► bấm "Bảo vệ tranh"
   └─► overlay "Đang thêm lớp bảo vệ..." (~1-2 phút trên CPU)
        └─► hiện ảnh trước/sau + cosine/SSIM/PSNR + nút Tải
             └─► (tùy chọn) "Kiểm tra độ bền" ─► bảng 9 phép tẩy nhiễu
```

## E. Điểm đáng nói khi phỏng vấn về phần web

1. **Tách lõi khỏi giao diện**: web chỉ là một "mặt tiền" mỏng gọi `pipeline.py`; CLI gọi cùng lõi. → dễ bảo trì, dễ test.
2. **Preset thay vì để người dùng chỉnh epsilon/steps**: giảm gánh nặng nhận thức cho họa sĩ.
3. **Chống cache ảnh** bằng query `?t=timestamp` — chi tiết nhỏ nhưng tránh bug "ảnh không cập nhật".
4. **Bất đồng bộ (async) + overlay**: vì xử lý lâu, UI báo trạng thái rõ ràng, không "đơ" im lặng.
5. **Bảo mật trung thực**: hiện chưa có auth, CORS mở — phù hợp demo, nhưng biết nêu ra là điểm cộng.
