# 🎯 Câu hỏi phỏng vấn tổng hợp xuyên dự án | Cross-Project Interview Questions

> Đây là **40 câu hay gặp nhất** không gắn riêng dự án nào — gồm câu hành vi (behavioral) và câu kỹ thuật chung. Mỗi câu có gợi ý trả lời + **dự án nên dẫn chứng**. Dùng để luyện nhanh phút chót, bổ trợ cho file câu hỏi riêng của từng dự án. (Song ngữ: Tiếng Việt trước, English below.)

---

# 🇻🇳 TIẾNG VIỆT

## A. Câu hành vi & giới thiệu (1–12)

**1. Giới thiệu về bản thân.**
30–60 giây: bạn là ai, học/làm gì, 1–2 dự án nổi bật hợp JD, và vì sao quan tâm vị trí này. Kết bằng "em đặc biệt thích [lĩnh vực khớp JD]".

**2. Dự án bạn tự hào nhất?**
Chọn dự án hợp JD nhất. Nêu: vấn đề → giải pháp kỹ thuật khó nhất → kết quả/số liệu → bài học. (totp-manager nếu bảo mật, art-shield nếu AI/ML, quan-quen-map nếu frontend/test.)

**3. Thử thách kỹ thuật khó nhất từng gặp?**
Dùng STAR. Câu mạnh: "font hiệu lực" (word-checker), nhiễu bền EoT 0/9→9/9 (art-shield), test code mật mã (totp-manager).

**4. Lúc bạn phải tự học công nghệ mới nhanh?**
OAuth/MSAL/Graph/IMAP (outlook-mail-reader) hoặc viXTTS/TTS (voice-clone-web). Nhấn: chia nhỏ vấn đề + có đường lui (fallback).

**5. Một quyết định kỹ thuật bạn đánh đổi (trade-off)?**
Lưu trữ 2 tầng (quan-quen-map: localStorage vs IndexedDB); ε lớn trong art-shield (bảo vệ mạnh hơn nhưng nhiễu dễ lộ); giới hạn 80MB khi quét log (token-counter: an toàn vs đầy đủ).

**6. Lúc bạn sai/thất bại và đã xử lý thế nào?**
Thành thật một lỗi thật + cách phát hiện + sửa + bài học. Vd: ban đầu đo art-shield bằng "đảo nhãn" → nhận ra sai → chuyển sang cosine.

**7. Điểm mạnh lớn nhất?**
Chọn 1 điểm có dẫn chứng: "em hiểu sâu thứ mình làm" (tự cài TOTP/PGD thay vì dùng thư viện) hoặc "em chú trọng chất lượng" (test, giảm false positive).

**8. Điểm yếu?**
Nêu điểm yếu thật + đang cải thiện. Tránh "em quá cầu toàn". Vd: "em hay sa đà vào tối ưu sớm; giờ em ưu tiên cho chạy được trước rồi đo mới tối ưu".

**9. Vì sao chọn công ty/vị trí này?**
Nối kỹ năng bạn có với thứ công ty làm. Đọc JD trước, chọn 1–2 dự án minh họa độ khớp.

**10. Bạn xử lý phản hồi/chỉ trích code thế nào?**
Coi review là để cải thiện sản phẩm, không phải cá nhân; hỏi rõ lý do, sửa, rút kinh nghiệm.

**11. Làm việc nhóm hay một mình tốt hơn?**
Linh hoạt; nêu rằng các dự án cá nhân giúp bạn tự chủ toàn bộ vòng đời, nhưng bạn sẵn sàng review/giao tiếp khi làm nhóm.

**12. Mục tiêu 1–3 năm tới?**
Trung thực + hợp hướng công ty: vững nền tảng, đóng góp được vào sản phẩm thật, học sâu một mảng (vd backend/AI).

## B. Kỹ thuật chung — ngôn ngữ & lập trình (13–24)

**13. Async/await dùng khi nào? Khác đa luồng?**
Async hợp I/O-bound (gọi LLM, mạng, file). Khác threading: async đơn luồng + vòng lặp sự kiện; threading nhiều luồng (ở Python bị GIL nên hợp I/O). Dẫn: agent-talk (async generator stream), outlook (job đa luồng).

**14. REST stateless nghĩa là gì? Lợi ích?**
Mỗi request tự đủ thông tin, server không giữ phiên → dễ scale ngang. Dẫn: agent-talk gửi state mỗi lượt.

**15. Giải thích vòng đời một HTTP request trong app của bạn.**
Client → API (validate bằng Pydantic) → xử lý/lõi → trả JSON/file → frontend render. Dẫn bất kỳ app FastAPI nào.

**16. OOP & SOLID — nêu một nguyên tắc bạn áp dụng.**
Single Responsibility: mỗi loại kiểm tra một module (word-checker); Dependency Inversion: engine pluggable (voice-clone-web).

**17. DRY là gì? Ví dụ thật?**
Don't Repeat Yourself: lõi dùng chung cho CLI + web (art-shield `pipeline.py`, word-checker runner).

**18. Cách bạn xử lý lỗi (error handling)?**
Bắt lỗi đúng tầng, thông báo rõ ràng cho người dùng, fail an toàn. Dẫn: token-counter (try/except quanh đọc file), voice-clone (phân biệt 503 chưa cài TTS vs 500).

**19. Lazy loading là gì? Vì sao dùng?**
Chỉ nạp tài nguyên nặng khi cần → khởi động nhanh. Dẫn: voice-clone (model viXTTS), art-shield (CLIP singleton).

**20. Bạn quản lý dữ liệu lớn/giới hạn tài nguyên thế nào?**
Giới hạn dung lượng, xử lý theo luồng, cache. Dẫn: token-counter (bỏ file >80MB, cap 2000 records), art-shield (giới hạn ảnh 512px).

**21. Đệ quy hữu ích khi nào?**
Duyệt cấu trúc cây không biết trước độ sâu. Dẫn: token-counter dò object usage đệ quy trong JSON.

**22. Regex bạn dùng để làm gì?**
Lọc/trích văn bản. Dẫn: token-counter (lọc dòng log), word-checker (kiểm tra định dạng).

**23. Cache/memoization là gì? Ví dụ?**
Nhớ kết quả để khỏi tính lại. Dẫn: token-counter (`lru_cache` cho encoder), art-shield (CLIP nạp 1 lần).

**24. Khác biệt cấu trúc dữ liệu (list/dict/set)?**
list có thứ tự đổi được; dict cặp khóa-giá-trị tra O(1); set khử trùng + kiểm tra thành viên O(1). Dẫn: dùng set fingerprint chống trùng (token-counter).

## C. Kỹ thuật chung — web, bảo mật, chất lượng (25–40)

**25. CORS là gì? Vì sao cần?**
Cơ chế trình duyệt kiểm soát gọi API khác origin. Các app mở `*` cho demo — nói thẳng production cần siết.

**26. Mã hóa vs hash khác nhau?**
Mã hóa hai chiều (giải được); hash một chiều. Dẫn: totp-manager (Fernet mã hóa, scrypt KDF), quan-quen-map (AES-GCM backup).

**27. Chống XSS thế nào?**
Escape nội dung động trước khi nhét vào innerHTML. Dẫn: token-counter (`escapeHtml` cho dữ liệu log không tin cậy).

**28. Xác thực (authentication) vs phân quyền (authorization)?**
Xác thực = bạn là ai (401); phân quyền = bạn được làm gì (403). Dẫn: outlook (OAuth/MSAL).

**29. Khi nào dùng SSE vs WebSocket?**
SSE một chiều server→client (hợp stream text); WebSocket hai chiều. Dẫn: agent-talk (stream phản hồi).

**30. Bạn test code thế nào? Loại test nào?**
Unit (một hàm), E2E (cả luồng), property-based (bất biến). Dẫn: totp-manager (RFC vectors + hypothesis), quan-quen-map (47 Playwright).

**31. Property-based testing là gì? Vì sao mạnh?**
Khai báo bất biến, thư viện sinh input ngẫu nhiên để phá. Mạnh hơn vài ca thủ công — dẫn totp-manager.

**32. Mock dùng để làm gì?**
Thay dịch vụ thật (LLM, fetch) để test nhanh/ổn định/offline. Dẫn: agent-talk, quan-quen-map.

**33. CI/CD là gì? Bạn đã dùng chưa?**
Tự chạy test khi push, chỉ deploy khi pass. Dẫn: quan-quen-map (GitHub Actions).

**34. Bạn đảm bảo chất lượng cho code "không thể sai" (mật mã/đo lường) thế nào?**
Dùng chuẩn chính thức + đo bằng tiêu chí đúng. Dẫn: totp-manager (RFC vectors), art-shield (cosine + robustness suite).

**35. Trung thực dữ liệu nghĩa là gì trong code của bạn?**
Không bịa số liệu khi không có. Dẫn: token-counter (báo credit thay token), art-shield (thừa nhận transfer kém).

**36. Bạn nghĩ về quyền riêng tư/đạo đức khi xây sản phẩm?**
Dẫn: token-counter (local-only, chỉ-đọc), voice-clone (consent bắt buộc), art-shield (định vị phòng thủ + cảnh báo giới hạn).

**37. Khi một API/endpoint chạy lâu, bạn xử lý UX thế nào?**
Async + báo trạng thái (overlay/spinner), không để UI "đơ" im lặng. Dẫn: art-shield (overlay 1–2 phút), voice-clone ("đang tải model").

**38. Giải thích một quyết định kiến trúc bạn tự hào.**
Tách lõi khỏi giao diện (art-shield pipeline dùng chung CLI+web), engine pluggable (voice-clone), kiến trúc tách lớp (word-checker).

**39. Nếu phải mở rộng dự án X cho 10.000 người dùng?**
Nêu nút thắt (CPU model nặng) + hướng (GPU/batch, hàng đợi job, cache, scale ngang nhờ stateless). Dẫn: art-shield/voice-clone (cần GPU), agent-talk (stateless dễ scale).

**40. Bạn có câu hỏi nào cho chúng tôi? (câu hỏi ngược)**
Luôn có sẵn 2–3 câu: quy trình review code, cách team đảm bảo chất lượng, kỳ vọng 3 tháng đầu. (Xem `KyNang_PhongVan/03_CAU_HOI_NGUOC.md`.)

---

# 🇬🇧 ENGLISH

## A. Behavioral & intro (1–12)

**1. Tell me about yourself.**
30–60s: who you are, what you study/do, 1–2 standout JD-relevant projects, why this role. End with "I'm especially keen on [area matching the JD]".

**2. The project you're proudest of?**
Pick the most JD-relevant. State: problem → hardest technical solution → result/numbers → lesson. (totp-manager for security, art-shield for AI/ML, quan-quen-map for frontend/testing.)

**3. Hardest technical challenge?**
Use STAR. Strong picks: "effective font" (word-checker), robust EoT noise 0/9→9/9 (art-shield), testing crypto (totp-manager).

**4. A time you self-taught a new tech fast?**
OAuth/MSAL/Graph/IMAP (outlook-mail-reader) or viXTTS/TTS (voice-clone-web). Emphasize: break it down + have a fallback.

**5. A technical trade-off you made?**
Two-tier storage (quan-quen-map); large ε in art-shield (stronger protection but more visible); the 80MB cap in log scanning (token-counter: safety vs completeness).

**6. A time you failed and how you handled it?**
Honestly state a real mistake + how you caught it + fixed it + the lesson. E.g. initially measured art-shield by "label flip" → realized it was wrong → switched to cosine.

**7. Your biggest strength?**
Pick one with evidence: "I understand deeply what I build" (hand-rolled TOTP/PGD instead of a library) or "I care about quality" (tests, false-positive reduction).

**8. A weakness?**
A real one + how you're improving. Avoid "I'm too perfectionist". E.g. "I tend to optimize too early; now I make it work first, measure, then optimize."

**9. Why this company/role?**
Connect your skills to what they do. Read the JD first, pick 1–2 projects showing the fit.

**10. How do you handle code-review feedback/criticism?**
Treat review as improving the product, not personal; ask for the rationale, fix, learn.

**11. Do you prefer teamwork or solo?**
Flexible; note solo projects gave you full lifecycle ownership, but you welcome review/communication on a team.

**12. Your goals for 1–3 years?**
Honest + aligned: solidify fundamentals, contribute to a real product, go deep in one area (e.g. backend/AI).

## B. General technical — language & programming (13–24)

**13. When async/await? vs multithreading?**
Async suits I/O-bound (LLM, network, file). Differs from threading: async is single-threaded + event loop; threading is multi-thread (in Python GIL makes it suit I/O). Cite: agent-talk (async-generator stream), outlook (multi-threaded job).

**14. What does REST stateless mean? Benefit?**
Each request is self-contained, the server keeps no session → easy horizontal scaling. Cite: agent-talk sends state each turn.

**15. Explain an HTTP request's lifecycle in your app.**
Client → API (Pydantic validation) → processing/core → return JSON/file → frontend renders. Cite any FastAPI app.

**16. OOP & SOLID — a principle you apply.**
Single Responsibility: one module per check (word-checker); Dependency Inversion: pluggable engine (voice-clone-web).

**17. What is DRY? A real example?**
Don't Repeat Yourself: a shared core for CLI + web (art-shield `pipeline.py`, word-checker runner).

**18. How do you handle errors?**
Catch at the right layer, give clear user messages, fail safely. Cite: token-counter (try/except around file reads), voice-clone (503 TTS-not-installed vs 500).

**19. What is lazy loading? Why use it?**
Load heavy resources only when needed → fast startup. Cite: voice-clone (viXTTS model), art-shield (CLIP singleton).

**20. How do you handle large data / resource limits?**
Size caps, streaming, caching. Cite: token-counter (skip >80MB files, cap 2000 records), art-shield (512px image limit).

**21. When is recursion useful?**
Traversing a tree of unknown depth. Cite: token-counter recursively detecting usage objects in JSON.

**22. What do you use regex for?**
Filtering/extracting text. Cite: token-counter (filter log lines), word-checker (format checks).

**23. What is caching/memoization? Example?**
Remember results to avoid recomputation. Cite: token-counter (`lru_cache` for the encoder), art-shield (CLIP loaded once).

**24. Differences between list/dict/set?**
list ordered-mutable; dict key-value O(1) lookup; set dedup + O(1) membership. Cite: a fingerprint set for dedup (token-counter).

## C. General technical — web, security, quality (25–40)

**25. What is CORS? Why needed?**
A browser mechanism controlling cross-origin API calls. The apps open `*` for demos — say plainly production should restrict it.

**26. Encryption vs hashing?**
Encryption is two-way (decryptable); hashing is one-way. Cite: totp-manager (Fernet encryption, scrypt KDF), quan-quen-map (AES-GCM backups).

**27. How to prevent XSS?**
Escape dynamic content before innerHTML. Cite: token-counter (`escapeHtml` for untrusted log data).

**28. Authentication vs authorization?**
Authentication = who you are (401); authorization = what you may do (403). Cite: outlook (OAuth/MSAL).

**29. When SSE vs WebSocket?**
SSE one-way server→client (good for text streaming); WebSocket two-way. Cite: agent-talk (streaming responses).

**30. How do you test code? Which types?**
Unit (a function), E2E (a whole flow), property-based (invariants). Cite: totp-manager (RFC vectors + hypothesis), quan-quen-map (47 Playwright).

**31. What is property-based testing? Why strong?**
Declare an invariant, the library generates random inputs to break it. Stronger than a few manual cases — cite totp-manager.

**32. What is mocking for?**
Replace real services (LLM, fetch) for fast, stable, offline tests. Cite: agent-talk, quan-quen-map.

**33. What is CI/CD? Have you used it?**
Auto-run tests on push, deploy only when passing. Cite: quan-quen-map (GitHub Actions).

**34. How do you assure quality for "cannot be wrong" code (crypto/measurement)?**
Use official standards + the right metric. Cite: totp-manager (RFC vectors), art-shield (cosine + robustness suite).

**35. What does "data honesty" mean in your code?**
Don't fabricate numbers you don't have. Cite: token-counter (report credits not tokens), art-shield (admit poor transfer).

**36. Do you think about privacy/ethics when building?**
Cite: token-counter (local-only, read-only), voice-clone (mandatory consent), art-shield (defensive positioning + limit warnings).

**37. When an API/endpoint is slow, how do you handle UX?**
Async + status feedback (overlay/spinner), never freeze silently. Cite: art-shield (1–2 min overlay), voice-clone ("loading model").

**38. Explain an architecture decision you're proud of.**
Separating core from UI (art-shield shared CLI+web pipeline), pluggable engine (voice-clone), layered architecture (word-checker).

**39. How would you scale project X to 10,000 users?**
Name the bottleneck (heavy model on CPU) + the path (GPU/batch, job queue, cache, horizontal scaling via statelessness). Cite: art-shield/voice-clone (need GPU), agent-talk (stateless scales easily).

**40. Do you have any questions for us? (reverse questions)**
Always have 2–3 ready: code-review process, how the team assures quality, expectations for the first 3 months. (See `KyNang_PhongVan/en/03_REVERSE_QUESTIONS.md`.)

---

> 💡 Mẹo: với mỗi câu kỹ thuật chung, **luôn kéo về một dự án thật** để chứng minh — đó là điểm khác biệt giữa "học thuộc" và "đã làm". | Tip: for each general-technical question, **always tie it back to a real project** — that's the difference between "memorized" and "actually did it".
