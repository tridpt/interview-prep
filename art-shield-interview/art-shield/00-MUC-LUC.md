# 📑 Mục lục — Bộ tài liệu phỏng vấn dự án Art Shield

Chào bạn! Đây là bộ tài liệu giúp bạn **hiểu cặn kẽ dự án Art Shield** để tự tin đi phỏng vấn, kể cả khi bạn chưa biết gì về AI hay nhiễu đối kháng. Đọc theo đúng thứ tự `00 → 08` là tối ưu nhất.

> Art Shield là công cụ **bảo vệ tranh của họa sĩ khỏi việc bị AI thu thập để huấn luyện**, bằng cách thêm một lớp "nhiễu đối kháng" (adversarial perturbation) mà mắt người gần như không thấy nhưng làm mô hình AI "nhìn sai" bức tranh.

## Thứ tự đọc

| File | Tên | Nội dung | Khi nào đọc |
|------|-----|----------|-------------|
| `00` | **Mục lục** (file này) | Bản đồ toàn bộ tài liệu | Đầu tiên |
| `01` | [Tổng quan dự án](01-tong-quan-du-an.md) | Dự án là gì, giải quyết vấn đề gì, công nghệ chính | Để nắm bức tranh lớn |
| `02` | [Khái niệm nền tảng](02-khai-niem-nen-tang.md) | CLIP, embedding, nhiễu đối kháng, PGD, EoT, PSNR/SSIM — giải thích từ số 0 | Bắt buộc, nền tảng lý thuyết |
| `03` | [Kiến trúc hệ thống](03-kien-truc-he-thong.md) | Các lớp, luồng dữ liệu, cách các module ghép với nhau | Sau khi nắm khái niệm |
| `04` | [Chi tiết lõi xử lý](04-loi-xu-ly-chi-tiet.md) | Đọc từng hàm: CLIP wrapper, attack PGD/EoT, perceptual mask, transforms, metrics | Phần "ruột" kỹ thuật |
| `05` | [Chi tiết web app](05-web-app-chi-tiet.md) | Backend FastAPI + frontend kéo-thả ảnh | Hiểu phần demo cho người dùng |
| `06` | [Tính năng nổi bật](06-tinh-nang-noi-bat.md) | Những điểm "ăn điểm" khi kể: EoT, masking tri giác, đo transfer | Để chọn điểm nhấn |
| `07` | [Câu hỏi phỏng vấn](07-cau-hoi-phong-van.md) | 60+ câu hỏi kèm trả lời mẫu, từ cơ bản đến khó | Luyện trả lời |
| `08` | [Cheat sheet](08-cheat-sheet.md) | Tóm tắt 1 trang để lướt phút chót | Trước giờ G |

## Mục tiêu sau khi đọc xong

1. **Kể được dự án trong 2 phút**: nó làm gì, cho ai, công nghệ gì.
2. **Giải thích được nguyên lý**: tại sao thêm nhiễu nhỏ lại làm AI nhìn sai, mà mắt người không thấy.
3. **Trả lời được câu hỏi vặn**: "tại sao chọn EoT?", "nhiễu có chuyển sang model khác không?", "giới hạn của giải pháp là gì?".
4. **Trung thực về hạn chế**: đây là demo nghiên cứu, không "miễn nhiễm 100%" — biết nói điều này đúng cách sẽ ghi điểm.

## Một câu để nhớ

> "Art Shield tối ưu một lớp nhiễu cực nhỏ trên ảnh sao cho **embedding của ảnh trong không gian CLIP bị đẩy ra thật xa ảnh gốc** — AI thấy một thứ khác hẳn, còn mắt người thì không. Để nhiễu sống sót qua nén/resize, tôi dùng kỹ thuật EoT; để giấu nhiễu cho đỡ lộ, tôi dùng bản đồ che tri giác."
