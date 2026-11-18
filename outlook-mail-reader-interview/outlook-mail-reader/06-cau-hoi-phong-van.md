# 6. Ngân hàng câu hỏi phỏng vấn & trả lời mẫu

70+ câu, chia nhóm, từ dễ đến khó. Mỗi câu có gợi ý trả lời. Đọc rồi tự diễn đạt lại.

> ⭐ = câu rất hay được hỏi.

---

## A. Tổng quan & động lực

**A1. ⭐ Giới thiệu ngắn gọn về dự án.**
> Outlook Mail Reader gộp hộp thư nhiều tài khoản Outlook/Microsoft 365 vào một chỗ. Mỗi tài khoản đăng nhập OAuth một lần (device code flow), sau đó app tự dùng refresh token (lưu mã hóa) để lấy mail mà không đăng nhập lại; không lưu mật khẩu. Backend FastAPI + MSAL + Microsoft Graph (và IMAP dự phòng), có unified inbox, tìm kiếm, đánh dấu đã đọc, đính kèm, import token đa luồng. Frontend JS thuần, render HTML mail trong iframe sandbox.

**A2. Vì sao làm dự án này?**
> Em hay dùng nhiều tài khoản Outlook và muốn gộp lại, đồng thời muốn thực hành tích hợp OAuth/identity thực tế — thứ gần như sản phẩm nào cũng cần. Em học được luồng OAuth, refresh token và lưu trữ token an toàn.

**A3. Đối tượng người dùng?**
> Người có nhiều tài khoản Outlook/M365 muốn xem gộp ở một nơi, riêng tư (chạy local, token ở máy mình).

**A4. ⭐ Dự án có an toàn/hợp lệ không?**
> Có: dùng OAuth chuẩn Microsoft, không lưu mật khẩu, mỗi tài khoản vẫn qua màn đăng nhập Microsoft thật một lần — nên chỉ dùng với tài khoản mình sở hữu/được phép. Quyền xin là Mail.ReadWrite (đọc + đánh dấu đã đọc, không gửi/xóa). Token lưu mã hóa.

---

## B. OAuth & MSAL (trọng tâm)

**B1. ⭐ Access token và refresh token khác nhau thế nào?**
> Access token ngắn hạn (vài chục phút) dùng gọi API; hết hạn phải lấy lại. Refresh token dài hạn, dùng để đổi lấy access token mới mà không cần đăng nhập lại — đó là thứ app lưu (mã hóa) để "đăng nhập 1 lần dùng mãi".

**B2. ⭐ Device code flow hoạt động ra sao? Vì sao dùng nó?**
> App xin Microsoft một mã ngắn + link; người dùng mở link, nhập mã, đăng nhập thật; app chạy nền poll tới khi xong rồi nhận token. Dùng vì app không có web redirect/trình duyệt nhúng — luồng này đơn giản và an toàn cho client công khai.

**B3. ⭐ MSAL giúp gì? Vì sao không tự viết OAuth?**
> MSAL (thư viện chính chủ Microsoft) lo device flow, tự refresh token, quản lý token cache đa tài khoản, xử lý hết hạn. Tự viết dễ sai và kém an toàn. Em chỉ bọc thêm mã hóa cache khi ghi đĩa.

**B4. Scope là gì? Dự án xin gì?**
> Phạm vi quyền. Dự án xin Mail.ReadWrite + User.Read, và offline_access để được cấp refresh token. Nếu chỉ muốn đọc thì đổi sang Mail.Read.

**B5. acquire_token_silent làm gì?**
> Lấy access token mới từ refresh token trong cache mà KHÔNG hỏi người dùng. Đây là cơ chế giúp các lần sau không phải đăng nhập lại.

**B6. Authority "common" nghĩa là gì?**
> Endpoint hỗ trợ cả tài khoản cá nhân (outlook.com/hotmail) lẫn tài khoản tổ chức (Microsoft 365). Nếu chỉ một tenant thì dùng tenant id thay cho "common".

**B7. Public client là gì? Vì sao phải bật "Allow public client flows"?**
> App native/desktop chạy ở máy người dùng nên không giữ được client secret an toàn → là public client. Device code flow cần bật cờ này trên Azure App Registration.

---

## C. Bảo mật & lưu trữ token

**C1. ⭐ Token lưu thế nào? Vì sao mã hóa?**
> MSAL serialize cache (chứa refresh token nhiều tài khoản) → mã hóa Fernet (AES) → ghi token_cache.bin. Vì refresh token là chìa khóa mail, không để plaintext dưới đĩa.

**C2. ⭐ Mã hóa mà khóa nằm cùng máy thì có ý nghĩa gì?**
> Thành thật: không chống được kẻ đã chiếm toàn quyền máy (khóa nằm ngay đó). Nhưng chống lộ ngẫu nhiên — backup nhầm, đẩy nhầm lên git, người khác đọc lướt file. Bảo vệ mạnh hơn cần khóa từ OS keychain/biến môi trường. Em ghi nhận đây là giới hạn.

**C3. Vì sao không lưu mật khẩu dù dòng import có cột password?**
> `_parse_import_line` tách 4 phần nhưng GÁN password vào biến `_password` rồi bỏ — chỉ dùng email/refresh_token/client_id. Không bao giờ lưu hay gửi password.

**C4. File khóa được bảo vệ ra sao?**
> `chmod 600` (chỉ chủ sở hữu đọc/ghi) trên POSIX; Windows bỏ qua an toàn. Cache hỏng/đổi khóa thì bỏ qua, người dùng đăng nhập lại.

**C5. Render HTML mail có rủi ro gì? Xử lý sao?**
> Mail HTML có thể chứa script/tracker → XSS. Em đặt body vào iframe `sandbox` (cô lập hoàn toàn, không chạy script, không thoát DOM cha); text thì bọc `<pre>` đã escapeHtml.

---

## D. Microsoft Graph & IMAP

**D1. ⭐ Graph API là gì? Dùng những endpoint nào?**
> REST API hợp nhất của Microsoft 365. Dùng /me/messages (list, có $top/$skip/$orderby/$select/$filter), /me/messages/{id} (chi tiết), $search (tìm), PATCH isRead (đánh dấu), /attachments, /mailFolders.

**D2. ⭐ Vì sao có cả IMAP? Khi nào dùng?**
> Một số refresh token chỉ dùng được IMAP (không Graph). Khi import, nếu Graph thất bại thì fallback sang IMAP (mode "imap"). Mọi thao tác đọc mail đều có hai nhánh Graph/IMAP theo mode tài khoản.

**D3. IMAP xác thực bằng OAuth thế nào (XOAUTH2)?**
> Thay vì mật khẩu, gửi chuỗi `user=<email>\x01auth=Bearer <access_token>\x01\x01` qua lệnh AUTHENTICATE XOAUTH2 — vẫn dùng OAuth token, không phải mật khẩu.

**D4. $filter và $search khác gì?**
> $filter lọc theo điều kiện chính xác (isRead eq false, hasAttachments eq true, ngày). $search tìm full-text, không kết hợp $orderby và cần header ConsistencyLevel: eventual.

**D5. Vì sao lọc người gửi làm phía client?**
> Graph $filter không hỗ trợ tìm gần đúng địa chỉ người gửi, nên sau khi tải trang mail, app lọc theo `sender` (so chuỗi name/address) ở client.

**D6. Mã id mail IMAP encode thế nào?**
> `imap:<base64(folder)>:<uid>` — vì id mail IMAP gắn với thư mục + UID; encode để truyền qua URL an toàn, decode lại khi cần (validate UID là số).

---

## E. Đa luồng & job

**E1. ⭐ Import job đa luồng hoạt động ra sao?**
> Đẩy mọi dòng vào Queue; khởi 1-5 worker thread lấy việc song song; mỗi worker import 1 dòng (gọi mạng kiểm chứng); Lock bảo vệ đếm processed/added/failed; frontend poll progress %; có nút hủy đặt cờ cancel_requested.

**E2. Race condition là gì? Bạn chống thế nào?**
> Nhiều luồng cùng sửa state job (processed++, results.append) có thể tranh chấp gây sai số. Em dùng threading.Lock bọc mọi lần đọc/ghi state dùng chung.

**E3. Vì sao worker là daemon thread?**
> Daemon thread tự kết thúc khi process chính thoát → không treo server. Job sống trong RAM, mất khi restart (chấp nhận được vì là tác vụ ngắn).

**E4. Frontend biết job xong bằng cách nào?**
> Poll GET /import-jobs/{id} mỗi 1.5s tới khi status là done/cancelled; khi đó tải toàn bộ kết quả (phân trang 200/lần) để hiện copy/retry dòng lỗi.

**E5. Đăng nhập cũng chạy nền — mẫu giống nhau?**
> Đúng. begin_login chạy thread chờ device flow, frontend poll /login/status. Cả hai là mẫu long-running task + polling.

---

## F. FastAPI & kiến trúc

**F1. ⭐ Mô tả kiến trúc backend.**
> config (cấu hình) ← store (mã hóa token) ← engine (logic: OAuth, Graph/IMAP, job) ← routes (HTTP) ← main (app + mount frontend). engine là singleton có Lock; state ở RAM + file mã hóa.

**F2. Vì sao lazy import engine trong từng route?**
> Tránh import vòng và tránh khởi tạo OutlookEngine (đọc file, dựng MSAL) quá sớm khi import module. Engine là singleton nên dùng chung một instance.

**F3. Backend có stateful không?**
> Có. Token cache + imported accounts lưu file mã hóa; login/import jobs lưu RAM. Khác app client-only — vì cần giữ phiên OAuth.

**F4. Xử lý lỗi thế nào?**
> Engine ném RuntimeError với thông điệp rõ; routes bắt và đổi thành HTTPException 400/404. Mỗi tài khoản/dòng import bọc try/except riêng để cô lập lỗi.

**F5. CORS allow_origins=* có vấn đề gì?**
> Tiện cho dev nhưng quá rộng nếu public. App này thiết kế chạy local. Nếu deploy công khai nên giới hạn origin + thêm xác thực (xem câu I3).

---

## G. Tính năng cụ thể

**G1. Unified inbox tải ra sao?**
> get_unified_inbox lặp từng tài khoản, gọi get_messages (Graph hoặc IMAP), gắn messages + has_more; lỗi 1 tài khoản thì gắn error riêng. Frontend phân trang theo từng tài khoản (skip = trang × PAGE).

**G2. Đánh dấu đã đọc cần gì?**
> Quyền Mail.ReadWrite. Graph: PATCH isRead. IMAP: STORE +FLAGS \Seen. Nếu token chỉ Mail.Read → báo lỗi cần ReadWrite.

**G3. ⭐ Check email live làm gì? Có chắc chắn không?**
> Validate format + tra MX qua Cloudflare DNS-over-HTTPS + đoán provider. Chỉ cho biết domain CÓ KHẢ NĂNG nhận mail; KHÔNG xác minh mailbox cụ thể tồn tại (vì chỉ có địa chỉ thì không thể chắc).

**G4. Health check account kiểm gì?**
> IMAP: mở thử INBOX. Graph: đọc thử 1 mail. Phân loại lỗi: token_expired / missing_scope / graph_error / imap_error...

**G5. Badge số chưa đọc lấy thế nào?**
> /unread-counts gọi inbox_unread_count mỗi tài khoản (Graph unreadItemCount hoặc IMAP search UNSEEN), lỗi thì để null (badge là phụ).

---

## H. Câu hỏi mở / tư duy

**H1. ⭐ Thử thách khó nhất?**
> (Gợi ý) Trừu tượng hai backend Graph/IMAP cho mọi thao tác mà vẫn gọn; hoặc làm import job đa luồng an toàn (Lock, cancel, progress) và dò scope khi import. Kể vấn đề → cách giải → kết quả.

**H2. ⭐ Có thêm thời gian sẽ cải thiện gì?**
> Lấy khóa mã hóa từ OS keychain/biến môi trường; thêm xác thực cho API local + siết CORS; lưu job ra store bền (sống qua restart); thêm soạn/gửi mail, thông báo mail mới, đọc mail bằng TTS (hướng dự án nêu); tách engine.py (1524 dòng) thành module nhỏ.

**H3. ⭐ App "vỡ" ở đâu khi nhiều tài khoản/nhiều mail?**
> get_unified_inbox gọi tuần tự từng tài khoản → nhiều tài khoản sẽ chậm (có thể song song hóa); Graph/IMAP có rate limit; IMAP mở/đóng kết nối mỗi thao tác (có thể pool lại). Token hết hạn thì health check + đổi token.

**H4. Quyết định kỹ thuật tự hào nhất?**
> Tích hợp OAuth/MSAL đúng chuẩn + lưu token mã hóa + render HTML an toàn. Đây là backend identity thực tế, không phải demo CRUD.

**H5. Trade-off lớn nhất?**
> Tính năng import refresh token rất mạnh (quản lý lô tài khoản) nhưng đòi hỏi nêu rõ ràng buộc đạo đức/pháp lý "chỉ tài khoản mình sở hữu". Chạy local + CORS mở → đơn giản nhưng cần siết khi public.

---

## I. Câu hỏi "code cụ thể"

**I1. _token_for() làm gì?**
> Điểm vào lấy access token cho mọi lời gọi Graph: nếu là imported graph → _refresh_imported_access_token (redeem refresh token); nếu MSAL → acquire_token_silent. Imported IMAP thì báo "không có Graph token".

**I2. _try_graph_import dò scope thế nào?**
> Lần lượt thử các bộ scope (Mail.Read/Mail.ReadWrite/...); mỗi bộ redeem token rồi GET /me/messages?$top=1; bộ nào 200 thì trả (kèm can_write); hết thì raise gộp lỗi.

**I3. save_cache chỉ ghi khi nào?**
> Khi cache.has_state_changed = True (MSAL báo cache đổi) → tránh ghi thừa.

**I4. _redeem_refresh_token cập nhật token mới chứ?**
> Có — refresh token có thể xoay vòng (rotation): nếu response trả refresh_token mới khác cũ thì lưu lại (có Lock) để lần sau dùng token mới.

**I5. Vì sao mở mail tự đánh dấu đã đọc nhưng "im lặng" nếu lỗi?**
> openMail gọi setRead(true, refresh=false); nếu token chỉ Mail.Read sẽ lỗi "Mail.Read" → bắt và bỏ qua (không báo phiền), vì đánh dấu là phụ.

**I6. FOLDER_MAP để làm gì?**
> Map tên thân thiện (inbox/sent/drafts/junk/deleted/archive) → well-known folder của Graph (inbox/sentitems/drafts/junkemail/deleteditems/archive).

---

## 🎤 5 câu hay gặp nhất — luyện trước tiên
1. A1 — Giới thiệu dự án.
2. B1/B2 — Access vs refresh token & device code flow.
3. B3 — MSAL & vì sao không tự viết OAuth.
4. C1/C2 — Lưu token mã hóa & giới hạn.
5. D2 — Vì sao có cả Graph và IMAP.

## ❓ Câu hỏi NÊN HỎI LẠI nhà tuyển dụng
- Sản phẩm tích hợp OAuth/SSO (Microsoft/Google) không, dùng thư viện gì?
- Team lưu trữ và xoay vòng token/secret thế nào (keychain/vault)?
- Có dùng background job/queue cho tác vụ dài không, công nghệ gì (Celery/RQ...)?
