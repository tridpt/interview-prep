# 01 — Tổng quan dự án Art Shield

## 1. Dự án này là gì? (giải thích như cho người chưa biết gì)

Hãy tưởng tượng bạn là một **họa sĩ**. Bạn đăng tranh lên mạng. Ngày nay có những công ty AI **cào (scrape)** hàng triệu bức tranh trên Internet về để **huấn luyện (train)** các mô hình vẽ tranh tự động. Hệ quả: AI có thể bắt chước phong cách của bạn mà bạn không hề đồng ý, không được trả tiền.

**Art Shield** giúp họa sĩ chống lại điều đó. Ý tưởng: trước khi đăng tranh, ta thêm vào ảnh một **lớp nhiễu cực nhỏ** (gọi là *nhiễu đối kháng* — adversarial perturbation). Lớp nhiễu này:

- **Mắt người gần như không thấy**: ảnh trông y hệt bản gốc.
- **Làm lệch đặc trưng của CLIP mục tiêu** trong thử nghiệm; chưa kiểm chứng được việc ngăn mô hình tạo ảnh học hay sao chép phong cách.

Đây chính là ý tưởng đứng sau các công cụ nổi tiếng như **Glaze** và **Nightshade** của Đại học Chicago. Art Shield là một bản **demo nghiên cứu** tái hiện kỹ thuật đó.

## 2. Nó giải quyết vấn đề gì?

| Vấn đề | Cách Art Shield xử lý |
|--------|----------------------|
| Tranh bị cào để train AI không xin phép | Thử thêm nhiễu đối kháng nhằm gây khó cho mô hình xử lý ảnh cụ thể; không bảo đảm bảo vệ mọi mô hình |
| Nhiễu phải vô hình với mắt người | Đo bằng PSNR/SSIM + giấu nhiễu vào vùng nhiều chi tiết (perceptual masking) |
| Kẻ thu thập có thể nén/resize để "tẩy" nhiễu | Huấn luyện nhiễu bằng EoT để nó sống sót qua các phép biến đổi |
| Người dùng không rành kỹ thuật | Web app kéo-thả ảnh, chọn mức "Nhẹ/Vừa/Mạnh" |

## 3. Các giai đoạn của dự án

Dự án được xây theo 3 giai đoạn rõ ràng (ghi trong README):

- **Giai đoạn 1 — Chứng minh kỹ thuật**: thêm nhiễu làm CLIP nhận sai, đồng thời ảnh gần như không đổi (PSNR/SSIM cao). File `protect.py`.
- **Giai đoạn 2 — Đo độ bền + nâng cấp**: kiểm tra nhiễu có sống sót qua nén/resize/làm mờ không (`robustness.py`); nâng cấp từ PGD ngây thơ (0/9 sống sót) lên **EoT** (9/9) và thêm **perceptual masking**.
- **Giai đoạn 3 — Web app**: giao diện tiếng Việt cho họa sĩ (`backend/` + `frontend/`).

## 4. Công nghệ chính

| Thành phần | Công nghệ | Vai trò |
|-----------|-----------|---------|
| Mô hình AI mục tiêu | **CLIP** (OpenAI, qua `transformers`) | "Mắt AI" mà ta đánh lừa |
| Học sâu / tối ưu | **PyTorch** | Tính gradient để tạo nhiễu |
| Thuật toán tạo nhiễu | **PGD** + **EoT** (tự cài) | Tối ưu lớp nhiễu đối kháng |
| Đo chất lượng ảnh | **PSNR, SSIM** (`scikit-image`) | Chứng minh nhiễu khó thấy |
| Web backend | **FastAPI** + `uvicorn` | API `/api/protect`, `/api/verify` |
| Web frontend | HTML/CSS/JS thuần | Kéo-thả ảnh, hiển thị kết quả |
| Xử lý ảnh | **Pillow (PIL)**, NumPy | Đọc/ghi, biến đổi ảnh |

## 5. Cấu trúc thư mục (rút gọn)

```
art-shield/
├── protect.py          # CLI: thêm nhiễu bảo vệ + báo cáo
├── robustness.py       # CLI: kiểm tra độ bền trước nén/resize/làm mờ
├── eval_transfer.py    # Thực nghiệm: nhiễu có "chuyển" sang model khác không
├── make_sample.py      # Tạo ảnh mẫu để thử
├── shield/             # LÕI dùng chung
│   ├── clip_model.py       # Bọc CLIP, encode ảnh/text, phân loại
│   ├── attack.py           # PGD + PGD-EoT + ensemble (tạo nhiễu)
│   ├── diff_transforms.py  # Biến đổi KHẢ VI cho EoT (mờ/resize/nén)
│   ├── transforms.py       # Biến đổi (PIL) để TEST độ bền
│   ├── perceptual.py       # Bản đồ che tri giác
│   ├── pipeline.py         # Lõi dùng chung cho CLI + web
│   └── metrics.py          # PSNR / SSIM
├── backend/            # Web app (FastAPI)
│   ├── main.py
│   └── config.py           # Preset Nhẹ/Vừa/Mạnh, đường dẫn
└── frontend/           # Giao diện web
    ├── index.html / app.js / style.css
```

## 6. Điểm khác biệt đáng kể

- **Tự cài thuật toán PGD và EoT từ đầu** bằng PyTorch (không dùng thư viện adversarial có sẵn) → chứng tỏ hiểu sâu cơ chế gradient.
- **Trung thực về giới hạn**: `eval_transfer.py` so sánh CLIP-B/32 với CLIP-B/16; **không chạy GPT-4o** và không đo hiệu quả chống huấn luyện mô hình tạo ảnh.
- **Tiêu chí đánh giá đúng**: dùng **cosine similarity với ảnh gốc** làm thước đo chính (ngưỡng < 0.5), không dựa vào việc "đảo nhãn phân loại" (vốn dễ gây hiểu lầm).

## 7. Pitch 30 giây

> "Art Shield là công cụ bảo vệ tranh họa sĩ khỏi việc bị AI cào đi train. Tôi thêm một lớp nhiễu cực nhỏ, vô hình với mắt người nhưng làm mô hình CLIP trích đặc trưng sai hẳn đi. Tôi tự cài PGD trên PyTorch để tối ưu nhiễu, dùng EoT để nhiễu sống sót qua nén/resize, và perceptual masking để giấu nhiễu. Tôi cũng có công cụ đo độ bền và đo khả năng 'chuyển' sang model khác — và thẳng thắn rằng giải pháp không tuyệt đối, đúng như Glaze/Nightshade."
