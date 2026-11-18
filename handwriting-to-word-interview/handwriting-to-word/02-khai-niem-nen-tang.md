# 02 — Khái niệm nền tảng

## Nét vector và ảnh raster

`HandwritingCanvas` lưu nét theo tọa độ và thời gian; vẽ lại lên bitmap canvas khi cần. Dữ liệu nét thuận tiện cho undo/redo và API handwriting, còn ảnh từ `toDataURL()` phục vụ nhận diện Vision. Không đánh đồng vector strokes với OCR từ ảnh chụp.

## Phân dòng và ứng viên

`HandwritingRecognizer.segmentLines()` phân nhóm nét theo khoảng Y và độ chồng lấp, có xử lý dấu phụ. Sau đó `recognize()` gọi API theo dòng. Heuristic có thể nhầm nét của hai dòng gần nhau; candidate bar cho phép người dùng sửa kết quả.

## Toán học

`MathEngine` phát hiện cấu trúc như phân số, số mũ, căn/tổng/tích phân từ vị trí nét, kết hợp bộ nhận diện ký tự. LaTeX mô tả công thức; KaTeX chỉ **render để xem**, không phải trình tạo Word equation. Gemini Vision là nhánh tùy chọn cho ảnh/nét khó.

## Tin cậy và riêng tư

OCR không chạy offline. Nét/ảnh gửi tới Google khi nhận diện; khi bật Gemini thì ảnh gửi tới dịch vụ Gemini. Key Gemini giữ trong `sessionStorage`, không gửi qua proxy OCR của app; tránh dùng tài liệu nhạy cảm và xem điều khoản dịch vụ trước khi dùng.
