# 7. Kiến thức CS nền tảng (Git, Docker, SOLID, độ phức tạp, mật mã)

Những thứ "ai cũng nên biết" và hay bị hỏi xen kẽ.

## 1. Độ phức tạp & cấu trúc dữ liệu
- **Big-O**: O(1) < O(log n) < O(n) < O(n log n) < O(n²). Sort tốt = O(n log n).
- **Mảng** (truy cập O(1), chèn giữa O(n)) · **hash map/set** (tra O(1) trung bình) · **stack/queue** (LIFO/FIFO) · **cây/đồ thị** (duyệt DFS/BFS).
- Mẹo: gặp "tìm/đếm/trùng" → nghĩ tới **hash**.

## 2. Git
- `add` → `commit` → `push`; nhánh: `branch`/`checkout`/`merge`.
- **merge vs rebase**: merge tạo commit gộp (giữ lịch sử thật); rebase viết lại lịch sử thẳng (sạch nhưng đừng rebase nhánh đã chia sẻ).
- **Pull request / code review**: quy trình hợp tác. `.gitignore` (bỏ file nhạy cảm — vd `.env`, `storage/vault.bin`).
- Conflict: xảy ra khi hai bên sửa cùng vùng → giải tay.

## 3. Docker (cap-cuu-101/agent-talk có Dockerfile)
- **Image** (khuôn) vs **Container** (thực thể đang chạy).
- `Dockerfile`: `FROM` (ảnh nền) → `COPY`/`RUN` (cài) → `CMD` (lệnh chạy).
- **Layer cache**: đặt bước cài dependencies TRƯỚC khi copy code → rebuild nhanh (code đổi nhiều, deps ít đổi).
- Truyền cấu hình qua biến môi trường (`-e LLM_API_KEY=...`).

## 4. SOLID & nguyên tắc thiết kế
- **S**ingle Responsibility: một lớp/module một trách nhiệm (word-checker: mỗi loại kiểm tra một module).
- **O**pen/Closed: mở để mở rộng, đóng để sửa (thêm kiểm tra = thêm hàm, không sửa lõi).
- **L**iskov / **I**nterface Segregation / **D**ependency Inversion.
- **DRY** (Don't Repeat Yourself): runner dùng chung cho CLI + web. **KISS**, **YAGNI**.
- **Separation of concerns**: đọc dữ liệu ≠ logic ≠ xuất kết quả (word-checker: document/checkers/report).

## 5. Process vs Thread vs Async
- **Process**: bộ nhớ riêng. **Thread**: chung bộ nhớ trong process (cẩn thận race condition → dùng **Lock**, vd outlook import job).
- **Async (event loop)**: một luồng nhưng nhả khi chờ I/O — hợp I/O-bound.
- Python GIL: threading hợp I/O-bound; CPU-bound nên multiprocessing.

## 6. Mật mã cơ bản (gắn totp-manager, quan-quen-map)
- **Hash** (SHA-256): một chiều, không giải ngược; dùng kiểm toàn vẹn.
- **HMAC**: hash kèm khóa bí mật để xác thực (TOTP dùng).
- **Đối xứng** (AES/Fernet): một khóa cho cả mã hóa & giải mã.
- **Bất đối xứng** (RSA): cặp khóa công khai/riêng tư.
- **KDF** (scrypt/PBKDF2): mật khẩu → khóa, cố tình tốn CPU/RAM chống brute-force. **Salt** chống rainbow table; **IV** để cùng dữ liệu ra bản mã khác.
- **Mã hóa-tại-chỗ (at rest)**: dữ liệu lưu đã mã hóa (vault.bin).

## 7. Cơ sở dữ liệu (cơ bản)
- **SQL** (quan hệ, ACID, join) vs **NoSQL** (linh hoạt, scale ngang).
- **Index** tăng tốc đọc, đánh đổi tốc độ ghi + dung lượng.
- **Transaction / ACID**: Atomicity, Consistency, Isolation, Durability.

## Câu hỏi mẫu
- "merge vs rebase?" · "Vì sao cài deps trước copy code trong Dockerfile?" · "SOLID — chữ S?" · "Hash vs mã hóa khác gì?" · "Race condition là gì, chống sao?" (Lock) · "Index đánh đổi gì?"

> 💡 Với mỗi khái niệm, chuẩn bị **một ví dụ từ dự án của bạn** — biến lý thuyết khô khan thành câu chuyện thật, ấn tượng hơn nhiều.
