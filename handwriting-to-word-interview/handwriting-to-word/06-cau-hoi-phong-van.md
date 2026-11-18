# 06 — Câu hỏi phỏng vấn

**1. OCR chạy offline không?** Không. Nhận diện chữ/nét đi qua Google Handwriting IME; chế độ toán heuristic cần recognizer cho ký tự. Soạn thảo và lưu nháp có thể chạy cục bộ sau khi tải app, nhưng không gọi đó là OCR offline.

**2. Vì sao lưu strokes thay vì chỉ lưu ảnh?** Tọa độ nét hỗ trợ chia dòng, hoàn tác, vẽ lại và gửi request dạng vector. Ảnh raster chỉ dùng cho nhánh Vision/xuất ảnh.

**3. Proxy giải quyết gì?** Frontend gọi đường dẫn ổn định `/api/handwriting`; Vite và Node server chuyển tiếp lên Google. Điều này giảm phụ thuộc vào CORS gọi trực tiếp, không biến API bên ngoài thành dịch vụ của mình.

**4. Toán 2D hoạt động ra sao?** Heuristic xét tương quan vị trí của nét (tử/mẫu, số mũ, v.v.), nhận diện ký tự và ghép LaTeX; có mẫu chèn nhanh và Gemini Vision tùy chọn. Công thức phức tạp có thể sai.

**5. `.docx` và PDF có tương đương màn hình không?** Không hoàn toàn. Word được tạo qua `docx`, công thức ghi LaTeX dạng text; PDF dùng hộp thoại in trình duyệt. Cần chuyển LaTeX thành OMML nếu muốn equation Word thật.

**6. Rủi ro riêng tư?** Nét/ảnh gửi tới dịch vụ OCR; nếu dùng Gemini, ảnh và key được dùng để gọi API. Không nhập dữ liệu nhạy cảm; xem chính sách của bên cung cấp.

**7. Test và CI?** `package.json` có build nhưng repo hiện chưa có thư mục test/CI cho luồng này. Không nhận công đã có E2E; ưu tiên test phân dòng, toán 2D, export và lỗi mạng.
