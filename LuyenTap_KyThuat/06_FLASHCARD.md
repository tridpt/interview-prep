# 6. Flashcard — Ôn nhanh hỏi/đáp

Che phần đáp án, tự trả lời. Lướt nhanh trước giờ G. (Q = hỏi, A = đáp ngắn.)

## JavaScript
- **Q:** `let` vs `const` vs `var`? **A:** block/block-không-gán-lại/function-tránh-dùng.
- **Q:** `==` vs `===`? **A:** ép kiểu vs nghiêm ngặt; dùng `===`.
- **Q:** Closure? **A:** hàm nhớ biến scope ngoài.
- **Q:** Event loop? **A:** đơn luồng + hàng đợi callback/microtask cho I/O.
- **Q:** Promise 3 trạng thái? **A:** pending / fulfilled / rejected.
- **Q:** localStorage vs IndexedDB? **A:** text/sync/5MB vs object/async/lớn.
- **Q:** Đọc streaming? **A:** `response.body.getReader()` + đọc chunk.
- **Q:** Event delegation? **A:** 1 listener ở container + `closest()`.

## Python
- **Q:** list/tuple/set/dict? **A:** đổi được-có-thứ-tự / bất biến / khử trùng / cặp khóa-giá-trị.
- **Q:** async khi nào? **A:** I/O-bound (mạng/file).
- **Q:** async generator? **A:** `async def` + `yield` → stream.
- **Q:** GIL? **A:** 1 luồng chạy bytecode → threading hợp I/O, CPU dùng multiprocessing.
- **Q:** decorator? **A:** hàm bọc hàm (`@app.post`, `@dataclass`).
- **Q:** dataclass để làm gì? **A:** struct dữ liệu gọn, tự sinh `__init__`.
- **Q:** Pydantic? **A:** validate request theo schema, sai → 422.

## Web / Real-time
- **Q:** 401 vs 403? **A:** chưa xác thực vs không có quyền.
- **Q:** 429? **A:** quá nhiều request (rate limit).
- **Q:** SSE vs WebSocket? **A:** một chiều vs hai chiều.
- **Q:** PWA cần gì? **A:** HTTPS + manifest + service worker.
- **Q:** Vòng đời SW? **A:** install → activate → fetch.
- **Q:** Chiến lược cache? **A:** cache-first / network-first / stale-while-revalidate.
- **Q:** REST stateless nghĩa? **A:** mỗi request tự đủ thông tin.
- **Q:** CORS? **A:** kiểm soát gọi API khác origin.

## Bảo mật / Mật mã (gắn totp-manager, quan-quen-map)
- **Q:** XSS chống sao? **A:** escape nội dung động trước innerHTML.
- **Q:** Mã hóa đối xứng vs bất đối xứng? **A:** cùng khóa / cặp khóa công-riêng.
- **Q:** Hash vs mã hóa? **A:** một chiều (không giải) vs hai chiều (giải được).
- **Q:** KDF (scrypt/PBKDF2) để làm gì? **A:** biến mật khẩu → khóa, cố tình chậm chống brute-force.
- **Q:** Salt? **A:** chuỗi ngẫu nhiên chống bảng tra (rainbow table).
- **Q:** AES-GCM? **A:** mã hóa đối xứng vừa mã hóa vừa xác thực toàn vẹn.
- **Q:** TOTP? **A:** OTP theo thời gian: HMAC(secret, time//30) + truncation.
- **Q:** "Không lưu khóa" nghĩa? **A:** khóa chỉ ở RAM; mất mật khẩu = mất dữ liệu.

## AI/ML & Speech (gắn art-shield, token-counter, voice-clone-web)
- **Q:** Embedding là gì? **A:** vector số mô tả nội dung (CLIP: 512 chiều); giống nhau → gần nhau.
- **Q:** Cosine similarity? **A:** đo độ giống hướng 2 vector; 1=giống, <0.5=đã "lệch xa".
- **Q:** Nhiễu đối kháng? **A:** nhiễu nhỏ có chủ đích (tìm bằng gradient) làm AI nhìn sai.
- **Q:** PGD? **A:** lặp gradient + chiếu nhiễu về [−ε,ε] để tạo nhiễu đối kháng.
- **Q:** EoT để làm gì? **A:** tối ưu nhiễu qua nhiều phép biến đổi → nhiễu bền với nén/resize.
- **Q:** STE? **A:** cho gradient chảy qua phép làm tròn (`x+(q-x).detach()`).
- **Q:** PSNR vs SSIM? **A:** sai khác pixel (dB) vs cảm nhận cấu trúc của mắt (0..1).
- **Q:** Transferability? **A:** khả năng nhiễu chuyển sang model khác (thường KÉM).
- **Q:** Token / Tokenizer / BPE? **A:** mẩu văn bản / bộ chuyển text↔token / gộp cặp ký tự thường gặp.
- **Q:** Encoding (tiktoken)? **A:** bảng từ vựng theo model (o200k_base, cl100k_base).
- **Q:** Token vs Credit? **A:** đơn vị model đọc (đếm được) vs đơn vị tính phí trừu tượng.
- **Q:** TTS vs voice cloning? **A:** văn bản→giọng / đọc bằng giọng học từ mẫu.
- **Q:** Zero-shot cloning? **A:** clone từ vài giây audio, không train lại.
- **Q:** Conditioning latents? **A:** speaker embedding + ngữ cảnh trích từ audio mẫu để "bắt chước" giọng.
- **Q:** Vì sao dùng WAV không webm cho dataset TTS? **A:** pipeline cần PCM thô không nén.
- **Q:** Fine-tune? **A:** huấn luyện tiếp model có sẵn trên dữ liệu mới (giọng của bạn).

## CS nền tảng
- **Q:** Big-O của sort? **A:** O(n log n).
- **Q:** Hash map tra thành viên? **A:** O(1) trung bình.
- **Q:** Process vs thread? **A:** không gian bộ nhớ riêng vs chung.
- **Q:** SOLID — S? **A:** Single Responsibility (một lớp một trách nhiệm).
- **Q:** DRY? **A:** Don't Repeat Yourself (vd runner dùng chung CLI + web).
- **Q:** Git: merge vs rebase? **A:** gộp giữ lịch sử / viết lại lịch sử thẳng.

## Testing
- **Q:** Unit vs E2E test? **A:** một hàm/đơn vị vs cả luồng người dùng.
- **Q:** Mock để làm gì? **A:** thay dịch vụ thật (LLM/fetch) để test nhanh, ổn định, offline.
- **Q:** Property-based testing? **A:** khai báo bất biến, thư viện sinh input ngẫu nhiên để phá.
- **Q:** CI/CD? **A:** tự chạy test khi push; chỉ deploy khi pass.

## Liên hệ dự án (tự hỏi)
- **Q:** Vì sao agent-talk stateless? **A:** state ở client gửi lên mỗi lượt → dễ scale.
- **Q:** Vì sao quan-quen-map dùng 2 tầng lưu trữ? **A:** ảnh nặng → IndexedDB, text → localStorage.
- **Q:** Vì sao word-checker bỏ qua công thức toán? **A:** giảm báo nhầm (false positive).
- **Q:** Vì sao art-shield đo bằng cosine, không phải "đảo nhãn"? **A:** nhãn có thể không đổi dù embedding đã lệch xa.
- **Q:** Vì sao token-counter báo "credit" thay vì token cho Kiro? **A:** Kiro không ghi token xuống log → trung thực, không bịa.
- **Q:** Vì sao voice-clone-web encode WAV ở client? **A:** để backend chỉ lưu file, không cần thư viện audio nặng.

> 💡 Ôn flashcard theo kiểu **active recall**: tự trả lời trước khi xem đáp án — nhớ lâu hơn đọc lại nhiều lần.
