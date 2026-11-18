# 05 — Tính năng nổi bật và đánh đổi

1. **Canvas có lịch sử nét:** lưu strokes thay vì chỉ ảnh, hỗ trợ undo/redo và chuyển đổi lại; đánh đổi là logic vẽ lại/tẩy phức tạp hơn.
2. **Tách dòng trước khi nhận diện:** `segmentLines()` giúp nhiều dòng không bị nhập thành một; chỉ là heuristic tọa độ nên không thay thế sửa thủ công.
3. **Hai đường nhận diện toán:** heuristic 2D không cần Gemini key nhưng vẫn dùng nhận diện ký tự qua handwriting API; Vision tùy chọn xử lý cấu trúc phức tạp hơn nhưng gửi dữ liệu ra ngoài.
4. **Biên tập + xuất bản:** typography, rich text và `.docx` được tạo ở trình duyệt; PDF thực chất là print-to-PDF, còn công thức trong Word là LaTeX thuần.
5. **Proxy dev/production:** cùng URL `/api/handwriting` cho frontend, nhưng phụ thuộc Google IME chứ không tự sở hữu mô hình OCR.
