# 4. System Design (thiết kế hệ thống)

Với fresher/junior, câu thiết kế thường nhẹ ("thiết kế một app rút gọn link", "thiết kế phần backend cho app X"). Quan trọng là **quy trình suy nghĩ rõ ràng**, không phải thuộc lòng kiến trúc khủng.

## Khung trả lời 6 bước
1. **Làm rõ yêu cầu**: chức năng (làm gì) + phi chức năng (bao nhiêu user, đọc/ghi nhiều hơn, cần realtime/offline?). **Luôn hỏi trước khi vẽ.**
2. **Ước lượng quy mô**: số user, QPS, dung lượng (thô thôi).
3. **API**: liệt kê các endpoint chính (method + path + in/out).
4. **Mô hình dữ liệu**: bảng/collection, quan hệ, index.
5. **Kiến trúc**: client → API → service → DB/cache; chỗ nào cần queue, cache, CDN.
6. **Bàn đánh đổi & nút thắt**: nhất quán vs sẵn sàng, mở rộng dọc/ngang, điểm chết.

## Các khái niệm cần biết
- **Mở rộng dọc (vertical)** = máy mạnh hơn; **ngang (horizontal)** = thêm máy + **load balancer**.
- **Stateless** giúp scale ngang dễ (mỗi request tự đủ thông tin — như agent-talk).
- **Cache** (Redis/CDN/cache trình duyệt) giảm tải; **đánh đổi**: dữ liệu có thể cũ.
- **DB**: SQL (quan hệ, ACID) vs NoSQL (linh hoạt, scale). **Index** tăng tốc đọc.
- **Queue** (hàng đợi) tách việc nặng/chậm ra xử lý nền (vd gọi LLM/gửi mail hàng loạt).
- **CAP**: phân tán không thể vừa nhất quán + sẵn sàng + chịu phân mảnh — phải chọn.

## Liên hệ với các dự án trong bộ (rất hữu ích để kể)
- **agent-talk**: backend **stateless** (state ở client gửi lên mỗi lượt) → dễ scale ngang. Nút thắt: gọi LLM chậm + rate limit → cần hàng đợi/giới hạn đồng thời nếu nhiều user.
- **quan-quen-map**: **client-only, không backend** → riêng tư, deploy tĩnh; đồng bộ đa thiết bị giải bằng GitHub Gist (đánh đổi: thủ công). Nếu lên quy mô lớn: chuyển lưu trữ sang DB/object storage.
- **totp-manager**: một người dùng, local-only → không cần phân tán; nếu multi-user phải thêm xác thực + tách dữ liệu per-user.
- **outlook-mail-reader**: phụ thuộc dịch vụ ngoài (Graph/IMAP) → cần **retry + fallback** (đã làm), và đa luồng cho tác vụ hàng loạt.

## Bài tập tự luyện (áp khung 6 bước)
- Thiết kế backend cho một app quản lý 2FA nhiều người dùng (gợi ý: từ totp-manager local → thêm auth, lưu vault per-user mã hóa).
- Thiết kế hệ thống kiểm tra file Word ở quy mô (gợi ý: từ word-checker CLI → API + hàng đợi xử lý file + lưu kết quả).

## Câu hỏi mẫu
- "Làm sao scale khi nhiều user?" → stateless + load balancer + cache + queue cho việc nặng.
- "SQL hay NoSQL?" → tùy quan hệ dữ liệu + nhu cầu nhất quán; nêu đánh đổi.
- "Xử lý dịch vụ ngoài chậm/giới hạn tần suất?" → retry có backoff + fallback + cache (vd fetchWithRetry trong quan-quen-map).

> 💡 Quan trọng nhất là **hỏi làm rõ + nói to suy luận + nêu đánh đổi**. Người phỏng vấn chấm cách bạn nghĩ, không phải một đáp án "đúng" duy nhất.
