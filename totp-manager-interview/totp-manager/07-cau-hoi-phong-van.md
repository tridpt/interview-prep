# 7. Ngân hàng câu hỏi phỏng vấn & trả lời mẫu

70+ câu, chia nhóm, từ dễ đến khó. Mỗi câu có gợi ý trả lời. Đọc rồi tự diễn đạt lại.

> ⭐ = câu rất hay được hỏi.

---

## A. Tổng quan & động lực

**A1. ⭐ Giới thiệu ngắn gọn về dự án.**
> OTP Vault là web app quản lý mã 2FA (TOTP) chạy cục bộ — nhập secret, nhận mã 6 số đổi mỗi 30 giây theo RFC 6238. Lưu nhiều tài khoản trong một kho mã hóa bảo vệ bằng mật khẩu chủ (scrypt + Fernet/AES), khóa chỉ ở RAM, tự khóa sau 5 phút. Backend FastAPI tự implement TOTP từ thư viện chuẩn; frontend vanilla JS (PWA) có quét QR. Có bộ test gồm test vector RFC 6238 và property-based testing.

**A2. Vì sao làm dự án này?**
> Em muốn một nơi an toàn trên máy tính để quản lý nhiều mã 2FA, và đây là cơ hội thực hành mật mã thực chiến: tự cài TOTP đúng chuẩn, mã hóa-tại-chỗ với KDF, và kiểm thử nghiêm túc.

**A3. Đối tượng người dùng?**
> Người dùng cá nhân muốn quản lý mã 2FA của chính mình trên máy tính, bảo mật bằng mật khẩu chủ.

**A4. ⭐ Dùng cho mục đích gì, có rủi ro lạm dụng không?**
> Dùng cho tài khoản của mình hoặc được phép quản lý. README ghi rõ đây KHÔNG phải công cụ nuôi tài khoản ảo hàng loạt. App local-only, một người dùng.

---

## B. TOTP & thuật toán

**B1. ⭐ TOTP hoạt động thế nào? Giải thích từng bước.**
> Decode secret base32 thành khóa → tính counter = floor(thời_gian/30) → HMAC(khóa, counter) ra digest → dynamic truncation (lấy offset từ 4 bit cuối, đọc 4 byte, bỏ bit dấu) → số % 10^digits, pad 0 → mã 6 số. Server và client cùng tính từ secret + thời gian nên ra cùng mã.

**B2. ⭐ TOTP khác HOTP thế nào?**
> HOTP (RFC 4226) dùng bộ đếm tăng dần; TOTP (RFC 6238) dùng counter = thời gian/period. TOTP là HOTP với counter dựa trên thời gian.

**B3. Vì sao secret là base32?**
> Base32 chỉ dùng A-Z, 2-7 — dễ gõ, ít nhầm lẫn ký tự (không có 0/O, 1/l). Là chuẩn của các app authenticator.

**B4. ⭐ Vì sao tự viết TOTP thay vì dùng pyotp?**
> TOTP chỉ là HMAC + truncation theo RFC — tự viết bằng hmac/hashlib chuẩn giúp giảm phụ thuộc, hiểu rõ thuật toán, và em kiểm chứng bằng test vector RFC 6238 chính thức. Phần mật mã khó (mã hóa kho) thì em dùng cryptography chuẩn chứ không tự chế.

**B5. Dynamic truncation là gì, để làm gì?**
> Bước RFC 4226 biến digest HMAC (20 byte) thành số ngắn: lấy 4 bit cuối làm offset, đọc 4 byte tại offset, bỏ bit dấu. Mục đích: ra mã ngắn dễ gõ mà vẫn phân bố đều, khó đoán.

**B6. normalize_secret xử lý gì?**
> Bỏ dấu cách/gạch, viết hoa, thêm '=' padding cho đủ bội số 8, rồi b32decode để validate. Idempotent. Vì người dùng hay copy secret lộn xộn.

**B7. seconds_remaining có thể trả 0 không?**
> Không — luôn trong [1, period]. Có test property-based kiểm với 200 thời điểm ngẫu nhiên.

**B8. Hỗ trợ mấy thuật toán?**
> SHA1 (mặc định), SHA256, SHA512 — qua dict `_ALGOS`. Sai thuật toán → TOTPError.

---

## C. Mã hóa & bảo mật (trọng tâm)

**C1. ⭐ Dữ liệu được mã hóa thế nào?**
> Mật khẩu chủ → scrypt(salt) → khóa 32 byte → Fernet (AES-128-CBC + HMAC-SHA256) mã hóa JSON danh sách. File vault.bin chỉ chứa {salt, ciphertext}.

**C2. ⭐ scrypt là gì, vì sao dùng?**
> Một KDF biến mật khẩu thành khóa, cố tình tốn nhiều bộ nhớ + CPU (N=32768, r=8, p=1) để chống brute-force. Salt 16 byte ngẫu nhiên chống rainbow table.

**C3. ⭐ Fernet là gì?**
> Một chuẩn mã hóa đối xứng an toàn của thư viện cryptography: AES-128-CBC + HMAC-SHA256, vừa mã hóa vừa xác thực toàn vẹn. Khóa = base64(32 byte từ scrypt).

**C4. ⭐ "Không lưu khóa" nghĩa là gì?**
> Khóa Fernet chỉ tồn tại trong RAM (Session.fernet) khi đã mở khóa. File chỉ có salt + ciphertext. Quên mật khẩu = mất dữ liệu, không có cửa hậu.

**C5. ⭐ Mở khóa xác thực mật khẩu thế nào mà không lưu hash?**
> unlock() dẫn khóa từ mật khẩu rồi thử fernet.decrypt(data). Giải mã thành công = đúng mật khẩu; InvalidToken = sai (raise WrongPassword → 401). Chính việc giải mã được LÀ bằng chứng.

**C6. Lấy được file vault.bin có nguy hiểm không?**
> Không, nếu mật khẩu mạnh: kẻ tấn công chỉ có salt + ciphertext, phải brute-force mật khẩu qua scrypt (rất chậm). Có test chứng minh file không chứa secret dạng rõ.

**C7. CSV có rủi ro gì?**
> CSV chứa secret ở dạng VĂN BẢN RÕ (chưa mã hóa). App cảnh báo rõ ở UI (confirm) và README, khuyên xóa file sau khi dùng. Đây là đánh đổi tiện lợi có chủ đích.

**C8. Backup mã hóa khác file vault thế nào?**
> Backup có salt riêng + mật khẩu riêng (do người dùng đặt khi xuất) nên tự chứa, khôi phục được trên máy khác. Vault dùng mật khẩu chủ của máy hiện tại.

---

## D. Phiên & auto-lock

**D1. ⭐ Auto-lock hoạt động thế nào?**
> Session lưu last_activity; is_unlocked() trả False nếu quá AUTO_LOCK_SECONDS (300s). Mỗi thao tác gọi touch() dời mốc. Quá hạn → khóa → API trả 401 → frontend hiện màn khóa.

**D2. Nếu file vault bị xóa khi phiên đang mở thì sao?**
> is_unlocked() kiểm file tồn tại; nếu mất → tự lock() và trả về khóa, tránh lỗi 500 khi đọc file. Có test cho ca này.

**D3. Vì sao dùng một Session toàn cục?**
> App chạy local, một người dùng — không cần multi-session/đăng nhập. Một biến module `session` dùng chung là đủ và đơn giản.

**D4. Frontend biết auto-lock thế nào?**
> Mỗi giây gọi /accounts; nếu server trả 401 (lỗi chứa "khóa"), client hiện lại màn khóa.

---

## E. API & FastAPI

**E1. Endpoint nào KHÔNG cần mở khóa? Vì sao?**
> /status, /setup, /unlock, /lock, /generate. generate chỉ tính mã từ secret truyền vào, không đụng kho nên an toàn để mở.

**E2. Quy tắc status code?**
> 200 OK; 400 request sai (mật khẩu <6, secret sai, CSV thiếu cột); 401 kho khóa / sai mật khẩu; 404 không tìm thấy tài khoản.

**E3. _code_payload bắt lỗi thế nào?**
> Bắt TOTPError khi tính mã cho một tài khoản và trả {code:None, error}; nhờ vậy một tài khoản secret hỏng không làm sập cả danh sách.

**E4. Pydantic dùng để làm gì?**
> Định nghĩa schema request (AddAccountRequest...); FastAPI tự validate JSON, sai kiểu trả 422.

**E5. Frontend được phục vụ thế nào?**
> main.py mount StaticFiles ở "/" (đặt cuối để không che /api), html=True để serve index.html.

---

## F. Frontend & PWA

**F1. ⭐ Quét QR hoạt động thế nào? Có gửi ảnh lên server không?**
> Không. Vẽ camera/ảnh lên canvas → getImageData → jsQR giải mã trong trình duyệt → chỉ gửi chuỗi otpauth:// lên server. Ảnh không rời máy.

**F2. Làm sao danh sách cập nhật mã liên tục?**
> setInterval(loadAccounts, 1000): mỗi giây gọi /accounts lấy mã + remaining mới, render lại vòng đếm ngược.

**F3. ⭐ Service worker có cache /api không? Vì sao?**
> Không bao giờ. Dữ liệu nhạy cảm (mã, danh sách) + cần tươi (mã đổi liên tục). SW dùng network-first cho app shell, bỏ qua hoàn toàn /api.

**F4. Chống XSS ở client?**
> escapeHtml mọi nội dung động (tên, ghi chú, lỗi) trước khi nhét vào innerHTML.

**F5. Kéo-thả sắp xếp lưu thế nào?**
> dragstart lưu draggedId; drop → reorderByDrop sắp lại mảng cục bộ + POST /accounts/reorder {ids} để server lưu (không chờ kết quả).

---

## G. Testing (điểm nhấn)

**G1. ⭐ Bạn test dự án thế nào?**
> 3 tầng pytest: test_totp (thuật toán), test_store (kho/mã hóa), test_api (end-to-end qua TestClient). Có test vector RFC 6238 và property-based testing bằng hypothesis.

**G2. ⭐ Test vector RFC 6238 là gì?**
> Bảng cặp (thuật toán, thời điểm → mã 8 số) chính thức trong chuẩn. Em assert generate_code khớp đúng cho SHA1/256/512 ở các mốc 59, 1111111109... → chứng minh implement đúng tuyệt đối.

**G3. ⭐ Property-based testing là gì? Bạn dùng ở đâu?**
> Khai báo một thuộc tính luôn đúng, hypothesis tự sinh hàng trăm input ngẫu nhiên để phá. Em dùng cho seconds_remaining (luôn ∈[1,period]) và round-trip parse(build(...)) bảo toàn tham số.

**G4. ⭐ Test có đụng dữ liệu thật không?**
> Không. conftest.py có fixture autouse chuyển VAULT_PATH sang thư mục tạm, giảm scrypt cho nhanh, reset session mỗi test. Không bao giờ ghi vào vault thật.

**G5. Test bảo mật nào đáng chú ý?**
> test_vault_encrypted_at_rest: thêm tài khoản rồi đọc file thô, assert secret + tên KHÔNG xuất hiện dạng văn bản rõ → chứng minh mã hóa-tại-chỗ.

**G6. "Req X.Y" trong test nghĩa là gì?**
> Dự án phát triển spec-driven (.kiro/specs): mỗi test ánh xạ tới một yêu cầu cụ thể trong đặc tả, đảm bảo phủ đủ yêu cầu.

---

## H. Câu hỏi mở / tư duy

**H1. ⭐ Thử thách khó nhất?**
> (Gợi ý) Cài TOTP đúng từng bit để khớp test vector RFC (endianness, truncation, padding base32); hoặc thiết kế mô hình mã hóa không lưu khóa mà vẫn xác thực được mật khẩu. Kể vấn đề → cách giải → kết quả.

**H2. ⭐ Có thêm thời gian sẽ cải thiện gì?**
> Thêm rate-limit thử mật khẩu; tăng scrypt N theo phần cứng; mã hóa từng tài khoản riêng thay vì cả kho (giảm phơi nhiễm khi giải mã); hỗ trợ multi-user nếu cần triển khai chung; bù thời gian (time skew) khi đồng hồ lệch.

**H3. Điểm yếu/giới hạn đã biết?**
> Local-only, một người dùng (không multi-user). CSV để secret rõ (đã cảnh báo). Khóa ở RAM nên ai có quyền đọc RAM tiến trình về lý thuyết thấy được khi đang mở khóa. Quên mật khẩu = mất dữ liệu.

**H4. Quyết định kỹ thuật tự hào nhất?**
> Tự cài TOTP đúng RFC và kiểm bằng test vector chính thức + property-based — cho thấy em hiểu thuật toán mật mã ở mức cài đặt, không chỉ ghép thư viện.

**H5. Trade-off lớn nhất?**
> "Không lưu khóa" → bảo mật cao nhưng quên mật khẩu là mất dữ liệu (không cửa hậu). Local-only → an toàn nhưng không truy cập từ xa được (phải tự đặt sau VPN nếu cần).

**H6. Nếu đồng hồ máy lệch thì sao?**
> TOTP phụ thuộc thời gian; lệch nhiều phút sẽ ra mã sai. Cải thiện: cho phép cửa sổ ±1 chu kỳ khi xác thực (server thật thường làm vậy); app này chỉ sinh mã nên dựa vào đồng hồ máy.

---

## I. Câu hỏi "code cụ thể"

**I1. struct.pack(">Q", counter) làm gì?**
> Đóng gói counter thành 8 byte unsigned big-endian — đúng định dạng message cho HMAC theo RFC.

**I2. & 0x7FFFFFFF để làm gì?**
> Bỏ bit cao nhất (bit dấu) của số 32-bit đọc từ digest, đảm bảo số dương — bước chuẩn của dynamic truncation.

**I3. Vì sao os.chmod(0o600) khi ghi vault?**
> Đặt quyền chỉ chủ sở hữu đọc/ghi (trên Unix) → người dùng khác trên máy không đọc được file kho.

**I4. import_backup bỏ trùng dựa trên gì?**
> Cặp (name, secret). Mục đã có thì bỏ qua; mục mới được cấp id mới (uuid). Tránh nhân đôi khi khôi phục.

**I5. reorder_accounts xử lý id rác/thiếu thế nào?**
> id không tồn tại bị bỏ qua; tài khoản không được nhắc tới giữ lại ở cuối theo thứ tự cũ → không bao giờ mất dữ liệu.

**I6. Vì sao thêm BOM (\ufeff) vào CSV?**
> Để Excel nhận đúng UTF-8 và hiển thị tiếng Việt không lỗi font.

---

## 🎤 5 câu hay gặp nhất — luyện trước tiên
1. A1 — Giới thiệu dự án.
2. B1/B4 — TOTP từng bước & vì sao tự viết.
3. C1/C4/C5 — Mã hóa, "không lưu khóa", unlock xác thực.
4. G2/G3 — Test vector RFC + property-based.
5. F1/F3 — Quét QR client & SW không cache /api.

## ❓ Câu hỏi NÊN HỎI LẠI nhà tuyển dụng
- Team xử lý dữ liệu nhạy cảm/mật mã thế nào (KDF, mã hóa-tại-chỗ)?
- Quy trình kiểm thử cho code bảo mật (test vector, property-based, fuzzing)?
- Có dùng spec-driven / yêu cầu hình thức trước khi code không?
