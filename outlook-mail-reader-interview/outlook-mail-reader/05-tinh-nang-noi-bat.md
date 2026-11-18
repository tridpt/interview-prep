# 5. Phân tích sâu các tính năng "ăn điểm"

Mỗi tính năng kèm: ý tưởng, cách làm, "vì sao hay".

---

## 5.1. OAuth 2.0 + MSAL: đăng nhập 1 lần, tự refresh ⭐⭐ (quan trọng nhất)

**Ý tưởng:** đọc mail mà không cần mật khẩu; đăng nhập một lần rồi dùng mãi.

**Cách làm:** device code flow qua MSAL → nhận refresh token → lưu vào token cache (mã hóa). Lần sau `acquire_token_silent()` tự đổi refresh token lấy access token mới. MSAL quản lý **nhiều tài khoản trong một cache**.

**Vì sao hay:** đây là **kiến thức backend cốt lõi** — gần như mọi sản phẩm doanh nghiệp đều cần "Đăng nhập bằng Microsoft/Google". Giải thích trôi chảy access token vs refresh token, scope, device code flow, và **silent token acquisition** là điểm cộng cực lớn.

**Có thể bị hỏi vặn:**
- *"Access token vs refresh token?"* → access ngắn hạn gọi API; refresh dài hạn đổi lấy access mới mà không đăng nhập lại.
- *"Device code flow để làm gì?"* → đăng nhập cho app không có trình duyệt nhúng/redirect: hiện mã + link, người dùng nhập ở trình duyệt khác.

---

## 5.2. Lưu trữ token an toàn (mã hóa Fernet) ⭐

**Ý tưởng:** refresh token là chìa khóa mail — không để plaintext.

**Cách làm:** serialize MSAL cache → `Fernet.encrypt` (AES) → ghi `token_cache.bin`; khóa ở `cache.key` (`chmod 600`); ghi có điều kiện `has_state_changed`. Imported accounts mã hóa riêng.

**Vì sao hay:** thể hiện ý thức bảo mật. **Quan trọng — nói được giới hạn:** khóa nằm cùng máy nên không chống được kẻ đã chiếm máy; nó chống lộ ngẫu nhiên (backup/git nhầm). Bảo vệ mạnh hơn cần OS keychain. Biết nêu hạn chế = trưởng thành.

---

## 5.3. Trừu tượng hai backend: Graph + IMAP ⭐

**Ý tưởng:** một số refresh token chỉ dùng được IMAP, không Graph → app phải hỗ trợ cả hai mà frontend không cần biết.

**Cách làm:** mỗi tài khoản có `mode` (graph/imap). **Mọi** thao tác (`get_messages`, detail, search, mark_read, attachments, folders, unread) đều kiểm tra mode rồi rẽ nhánh: Graph (REST + Bearer token) hoặc IMAP (XOAUTH2 + SELECT/SEARCH/FETCH + parse `email`).

**Vì sao hay:** đây là **lớp adapter/strategy** thực thụ — cùng một interface, hai cách hiện thực. Còn cho thấy hiểu cả REST API hiện đại lẫn giao thức IMAP truyền thống (XOAUTH2, UID, FLAGS `\Seen`).

---

## 5.4. Dò scope khi import token ⭐

**Ý tưởng:** một refresh token có thể có quyền `Mail.Read` hoặc `Mail.ReadWrite` — app cần tự biết để hiện đúng và chặn thao tác ghi nếu thiếu quyền.

**Cách làm:** `_try_graph_import` lần lượt thử các bộ scope (`_graph_import_scope_options`), với mỗi bộ gọi `GET /me/messages?$top=1`; bộ nào trả 200 thì dùng, ghi `graph_can_write`. Khi `mark_read` mà token chỉ Mail.Read → báo lỗi rõ "cần Mail.ReadWrite".

**Vì sao hay:** xử lý thực tế "không biết trước token có quyền gì" — thông minh và phòng thủ.

---

## 5.5. Import job đa luồng có tiến độ & hủy ⭐

**Ý tưởng:** import lô lớn refresh token (mỗi cái phải gọi mạng kiểm chứng) nhanh và có thể theo dõi/hủy.

**Cách làm:** `Queue` chứa dòng việc; 1-5 **worker thread** lấy việc song song; `threading.Lock` bảo vệ đếm processed/added/failed; frontend poll progress %, log phân trang; nút Hủy đặt cờ `cancel_requested` để worker tự dừng; cuối job tự đặt done/cancelled. Có **copy/retry dòng lỗi**.

**Vì sao hay:** mẫu **producer/consumer + job tracking** kinh điển, làm thủ công bằng thư viện chuẩn — cho thấy hiểu **concurrency, race condition, lock**.

---

## 5.6. Render HTML mail an toàn bằng iframe sandbox ⭐

**Ý tưởng:** mail HTML từ người lạ có thể có script/tracker độc.

**Cách làm:** đặt body vào `<iframe sandbox>` qua `srcdoc` → cô lập hoàn toàn (không script, không thoát DOM cha). Text thì `<pre>` đã escape.

**Vì sao hay:** một quyết định bảo mật rõ ràng, đúng best practice cho việc hiển thị nội dung không tin cậy.

---

## 5.7. Cô lập lỗi theo tài khoản ⭐

**Ý tưởng:** một tài khoản hết token/lỗi mạng không được kéo sập cả unified inbox.

**Cách làm:** `get_unified_inbox`/`search_messages` bọc try/except **từng tài khoản**, lỗi thì gắn `error` cho riêng tài khoản đó, các tài khoản khác vẫn trả mail bình thường.

**Vì sao hay:** tư duy **resilience** và UX — hỏng cục bộ, không hỏng toàn cục.

---

## 5.8. Check email bằng DNS/MX (DNS-over-HTTPS)

**Ý tưởng:** kiểm tra một địa chỉ email có khả năng nhận mail không, **mà không gửi mail thử**.

**Cách làm:** validate format → tra MX qua **Cloudflare DoH** → có MX = domain nhận được mail; đoán provider từ tên MX; xử lý IDN. Nêu rõ **không xác minh mailbox cụ thể**.

**Vì sao hay:** dùng DNS-over-HTTPS để tra MX không cần thư viện DNS; trung thực về giới hạn (chỉ kiểm domain, không kiểm mailbox).

---

## 5.9. Bảng tổng hợp "tính năng ↔ khái niệm kỹ thuật" (học thuộc)

| Tính năng | Khái niệm bạn thể hiện |
|-----------|------------------------|
| OAuth + MSAL | OAuth 2.0, access/refresh token, scope, device code flow, silent refresh |
| Token mã hóa | Fernet/AES, quản lý khóa, giới hạn bảo mật |
| Graph + IMAP | adapter/strategy, REST + OData, IMAP XOAUTH2 |
| Dò scope | xử lý quyền không biết trước, phòng thủ |
| Import job đa luồng | threading, Queue, Lock, race condition, job tracking |
| iframe sandbox | bảo mật XSS nội dung không tin cậy |
| Cô lập lỗi | resilience, try/except theo đơn vị |
| Check email DNS/MX | DNS-over-HTTPS, MX record, IDN |

> 💡 **Chiến lược kể chuyện:** nếu được hỏi "điểm em tự hào nhất?", chọn **OAuth/MSAL** (kiến thức identity cốt lõi, giá trị nhất) hoặc **trừu tượng Graph+IMAP** (thiết kế adapter). Cả hai cho thấy bạn làm được tích hợp thực tế phức tạp.
