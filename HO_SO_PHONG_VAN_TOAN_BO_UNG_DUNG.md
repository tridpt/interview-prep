# Hồ sơ phỏng vấn: 12 ứng dụng trong portfolio

> Dùng khi trao đổi với nhà tuyển dụng hoặc phỏng vấn kỹ thuật. Đây là **12 dự án độc lập**, không phải 12 thành phần của một ứng dụng chung. Nội dung dưới đây dựa trên mã nguồn và README trong workspace này; hãy thay phần trong ngoặc vuông bằng thông tin, vai trò và kết quả thực tế của bạn trước khi gửi CV.

## 1. Mở đầu trong 60 giây

> "Em có portfolio gồm 12 ứng dụng giải quyết các bài toán khá khác nhau: web chạy offline, backend Python, bảo mật, xử lý tài liệu và tích hợp AI. Em thường bắt đầu từ nhu cầu người dùng, chọn kiến trúc vừa đủ, rồi kiểm tra những rủi ro quan trọng nhất. Với vị trí **[tên vị trí]**, hai dự án phù hợp nhất là **[dự án A]** và **[dự án B]**: dự án A cho thấy **[kỹ năng khớp JD]**, còn dự án B cho thấy **[kỹ năng bổ sung]**. Em có thể demo luồng người dùng và đi sâu vào quyết định kỹ thuật hoặc phần code mà anh/chị quan tâm."

Không đọc tên cả 12 dự án trong phần mở đầu. Chỉ kể sâu **1-2 dự án liên quan JD**, giữ các dự án còn lại làm bằng chứng khi được hỏi thêm. Nếu làm cùng người khác hoặc dùng AI hỗ trợ, hãy nói chính xác phần bạn thiết kế, viết, kiểm thử và phần cần kiểm tra lại.

## 2. Chọn dự án theo vị trí

| Vị trí ứng tuyển | Dự án nên kể trước | Bằng chứng nên mở |
|---|---|---|
| Frontend / PWA | Quán Quen + AI Explorer hoặc Cấp Cứu 101 | UI, state, lưu cục bộ, service worker, kiểm thử giao diện |
| Backend Python / API | OTP Vault + Agent Talk hoặc Word Checker | API, lớp xử lý, validation, test, xử lý lỗi |
| Fullstack | OTP Vault + Quán Quen hoặc Agent Talk | Luồng UI -> API/dữ liệu, quyết định lưu trữ, test |
| AI / LLM | Agent Talk + Red-Team Lab | Orchestration, streaming, ranh giới tin cậy; phân biệt mô phỏng và LLM thật |
| AI/ML / computer vision | Art Shield + Agent Talk | Gradient/PGD/EoT, phép đo, giới hạn transfer giữa mô hình |
| Speech / audio | Voice Clone Web + Agent Talk | Pipeline thu âm -> WAV -> model -> audio, consent |
| Bảo mật / tích hợp hệ thống | OTP Vault + Outlook Mail Reader | RFC 6238, KDF/mã hóa, OAuth, vòng đời token |
| Tooling / xử lý dữ liệu | ContractShield + Word Checker hoặc AI Usage Monitor | Native OOXML/PDF parser, LCS word-diff, PII Vault, SHA-256 seal, E2E test |
| LegalTech / Document AI | ContractShield & Redline Studio + Agent Talk | Rule engine song ngữ VI/EN + Custom Playbook, mô phỏng tiền phạt What-If, OCR offline |

Xem thêm [bảng so sánh theo JD](SO_SANH_DU_AN.md), [câu hỏi xuyên dự án](CAU_HOI_TONG_HOP.md) và [lộ trình ôn tập](LO_TRINH_ON_TAP.md).

## 3. Hồ sơ từng ứng dụng

Mỗi mục đủ để kể trong khoảng 30-60 giây. Khi phỏng vấn kỹ thuật, mở file code được dẫn thay vì cố nhớ mọi hàm.

### Agent Talk - điều phối hội thoại nhiều AI

- **Giá trị:** cho nhiều agent có tính cách riêng trao đổi theo lượt, người dùng quan sát và xen vào; có streaming, tóm tắt ngữ cảnh và chấm điểm.
- **Quyết định kỹ thuật:** frontend giữ vòng lặp hội thoại; FastAPI dựng ngữ cảnh từng request và gọi LLM bên ngoài. Mở [API](../agent-talk/backend/main.py), [lớp LLM](../agent-talk/backend/llm.py), [frontend](../agent-talk/frontend/app.js).
- **Demo:** cấu hình ít nhất hai agent, chọn chủ đề, chạy vài lượt và xen vào. Cần API key riêng; test mock LLM có thể chạy không tốn key. Cách chạy ở [README](../agent-talk/README.md).
- **Nói đúng giới hạn:** app phụ thuộc chất lượng/chi phí provider; endpoint chia sẻ phiên hiện chưa có xác thực, chỉ phù hợp môi trường cục bộ. [Bộ ôn chi tiết](agent-talk-interview/agent-talk/00-MUC-LUC.md).

### AI Explorer - bảo tàng AI tương tác

- **Giá trị:** giải thích khái niệm AI bằng 26 phòng tương tác, lộ trình học, quiz, song ngữ và PWA.
- **Quyết định kỹ thuật:** mỗi phòng là ES module độc lập; router, i18n và tiến trình học do browser quản lý. Mở [điều phối](../ai-explorer/app.js), [một phòng mẫu](../ai-explorer/rooms/tokenizer.js), [PWA](../ai-explorer/pwa.js).
- **Demo:** mở một phòng, thao tác, làm quiz, đổi ngôn ngữ và thử offline sau khi đã cache. Cần HTTP server; xem [README](../ai-explorer/README.md).
- **Nói đúng giới hạn:** đây là mô phỏng/giáo dục khái niệm; một số tính năng như webcam phụ thuộc quyền trình duyệt. [Bộ ôn chi tiết](ai-explorer-interview/ai-explorer/00-MUC-LUC.md).

### Art Shield - thí nghiệm nhiễu đối kháng cho ảnh

- **Giá trị:** thử tạo nhiễu làm lệch embedding của mô hình CLIP mục tiêu, đồng thời đo mức thay đổi ảnh và độ bền qua nén/resize.
- **Quyết định kỹ thuật:** PGD/EoT và perceptual masking trong [thuật toán](../art-shield/shield/attack.py), [pipeline](../art-shield/shield/pipeline.py); giao diện qua [FastAPI](../art-shield/backend/main.py).
- **Demo:** chuẩn bị ảnh mẫu, chạy pipeline hoặc web UI, so ảnh trước/sau và đọc PSNR/SSIM cùng phép thử độ bền. Lần đầu tải model và có thể xử lý lâu; xem [README](../art-shield/README.md).
- **Nói đúng giới hạn:** kết quả trên CLIP được thử nghiệm **không chứng minh** chặn mọi mô hình học tranh hay sao chép phong cách. [Bộ ôn chi tiết](art-shield-interview/art-shield/00-MUC-LUC.md).

### Cấp Cứu 101 - học sơ cứu trên PWA

- **Giá trị:** mini-game ghi nhớ sơ cứu, chế độ tra cứu nhanh, âm thanh/giọng đọc và dùng offline sau khi cache.
- **Quyết định kỹ thuật:** HTML/CSS/JS thuần, [logic trò chơi](../cap-cuu-101/app.js), [service worker](../cap-cuu-101/sw.js), lưu tiến trình trên thiết bị.
- **Demo:** chơi một màn, mở chế độ tra cứu, đổi ngôn ngữ, thử giao diện di động. Chạy bằng HTTP server theo [README](../cap-cuu-101/README.md).
- **Nói đúng giới hạn:** nội dung là công cụ học tập, không thay đào tạo/cấp cứu chuyên môn. [Checklist rà soát y tế](../cap-cuu-101/MEDICAL_REVIEW.md) hiện chưa có người duyệt ký tên; đừng giới thiệu là hướng dẫn đã được chuyên gia phê duyệt. [Bộ ôn chi tiết](cap-cuu-101-interview/cap-cuu-101/00-MUC-LUC.md).

### InkToWord Studio - viết tay sang tài liệu

- **Giá trị:** vẽ nét trên canvas, nhận diện chữ viết tay tiếng Việt, biên tập rồi xuất `.docx` hoặc in ra PDF.
- **Quyết định kỹ thuật:** tách [canvas](../handwriting-to-word/src/canvas.js), [nhận diện](../handwriting-to-word/src/recognizer.js), [xuất Word](../handwriting-to-word/src/docx-export.js) và [toán](../handwriting-to-word/src/math-engine.js).
- **Demo:** viết một câu ngắn, chọn gợi ý nhận diện, chỉnh trong editor và xuất Word. Chạy Vite theo [README](../handwriting-to-word/README.md); nhận diện cần mạng.
- **Nói đúng giới hạn:** OCR không hoàn toàn offline; nhận diện toán có thể cần sửa LaTeX; công thức trong `.docx` là chuỗi LaTeX, chưa phải equation Word. Repo hiện chưa có test tự động riêng. [Bộ ôn chi tiết](handwriting-to-word-interview/handwriting-to-word/00-MUC-LUC.md).

### Outlook Mail Reader - hộp thư đa tài khoản

- **Giá trị:** gom hộp thư Outlook/Microsoft 365, tìm kiếm, xem nội dung và xử lý token/tài khoản qua OAuth.
- **Quyết định kỹ thuật:** [API](../outlook-mail-reader/backend/routes.py), [engine Graph/IMAP](../outlook-mail-reader/backend/engine.py), [lưu token](../outlook-mail-reader/backend/store.py); import lớn dùng job và nhiều luồng.
- **Demo:** dùng **tài khoản thử nghiệm được phép truy cập**, đăng nhập device code rồi xem inbox; có thể trình bày code/screenshot nếu chưa cấu hình Azure. Xem [README](../outlook-mail-reader/README.md).
- **Nói đúng giới hạn:** kiểm tra email bằng format/DNS/MX không chứng minh hộp thư tồn tại; khóa mã hóa token lưu trên cùng máy không bảo vệ khi máy đã bị chiếm. Không hiển thị token hoặc thư thật khi phỏng vấn. [Bộ ôn chi tiết](outlook-mail-reader-interview/outlook-mail-reader/00-MUC-LUC.md).

### Quán Quen - bản đồ địa điểm cá nhân

- **Giá trị:** lưu/quản lý quán ăn theo gu, tìm/lọc, ảnh, lịch trình, sao lưu và trải nghiệm PWA.
- **Quyết định kỹ thuật:** [logic ứng dụng](../quan-quen-map/app.js) phối hợp Leaflet, localStorage/IndexedDB; [service worker](../quan-quen-map/sw.js) cache app shell; [Playwright](../quan-quen-map/tests/app.spec.js) kiểm tra luồng UI.
- **Demo:** thêm quán, chấm điểm, lọc, xem ảnh/lịch trình rồi cho thấy backup. Chạy HTTP server theo [README](../quan-quen-map/README.md).
- **Nói đúng giới hạn:** thao tác bản đồ, tìm địa điểm và định tuyến ngoài mạng vẫn phụ thuộc OSM/Overpass/OSRM; offline chủ yếu dành cho giao diện và dữ liệu đã lưu. [Bộ ôn chi tiết](quan-quen-map-interview/quan-quen-map/00-MUC-LUC.md).

### Red-Team Lab - học prompt injection

- **Giá trị:** các màn trực tiếp và gián tiếp (RAG) giúp quan sát ranh giới tin cậy, bộ lọc, đầu vào không đáng tin và cách phòng thủ.
- **Quyết định kỹ thuật:** [engine trực tiếp](../red-team-lab/engine.js), [engine gián tiếp](../red-team-lab/indirect.js), [test](../red-team-lab/tests/engine.test.mjs) chạy trong Node; mặc định không cần backend/API key.
- **Demo:** thử một payload đơn giản, xem phản hồi, giải thích vì sao qua/chặn và bài học đội xanh. Chạy HTTP server theo [README](../red-team-lab/README.md).
- **Nói đúng giới hạn:** các màn chính dùng **simulator tất định**, không phải phép đo khả năng jailbreak một LLM thật. Sandbox gọi LLM thật là tùy chọn riêng. [Bộ ôn chi tiết](red-team-lab-interview/red-team-lab/00-MUC-LUC.md).

### AI Usage Monitor - theo dõi token/credit local

- **Giá trị:** quét metadata usage từ log công cụ AI trên máy, đồng thời đếm token của văn bản người dùng dán bằng tokenizer.
- **Quyết định kỹ thuật:** [scanner](../token-counter/backend/usage_scanner.py) xử lý nhiều schema/nguồn và [API](../token-counter/backend/main.py) cung cấp số liệu cho [UI](../token-counter/frontend/app.js).
- **Demo:** dùng một JSON usage mẫu hoặc đoạn văn bản mẫu; tránh chiếu log riêng tư. Cách chạy ở [README](../token-counter/README.md).
- **Nói đúng giới hạn:** credit/quota không thể quy đổi chính xác thành token nếu log không có metadata. Scanner chỉ phản ánh dữ liệu local hiện có. [Bộ ôn chi tiết](token-counter-interview/token-counter/00-MUC-LUC.md).

### OTP Vault - quản lý mã xác thực TOTP

- **Giá trị:** sinh mã RFC 6238, lưu tài khoản trong kho mã hóa bằng mật khẩu chủ, nhập QR/URI và sao lưu.
- **Quyết định kỹ thuật:** [TOTP](../totp-manager/backend/totp.py) dùng HMAC thư viện chuẩn; [kho](../totp-manager/backend/store.py) dùng scrypt + Fernet; [API](../totp-manager/backend/main.py) chỉ phục vụ local. [Test](../totp-manager/tests/test_totp.py) có vector RFC.
- **Demo:** dùng secret **giả**, so mã với test vector/đồng hồ, khóa/mở kho và xem bản backup mã hóa. Cách chạy ở [README](../totp-manager/README.md).
- **Nói đúng giới hạn:** quên mật khẩu chủ thì không giải mã lại được; CSV export chứa secret dạng rõ; đây là app local, không có auth đa người dùng để mở Internet. [Bộ ôn chi tiết](totp-manager-interview/totp-manager/00-MUC-LUC.md).

### Voice Clone Web - tổng hợp giọng tiếng Việt

- **Giá trị:** thu/tải giọng mẫu, chuẩn bị dataset và dùng viXTTS đọc văn bản theo giọng tham chiếu.
- **Quyết định kỹ thuật:** [API](../voice-clone-web/backend/main.py), [engine](../voice-clone-web/backend/tts_engine.py), [WAV recorder](../voice-clone-web/frontend/wav-recorder.js); engine có thể thay bằng implementation khác.
- **Demo:** chỉ dùng giọng của bạn hoặc người đã đồng ý rõ ràng; thử câu ngắn và cho xem luồng từ file WAV đến audio đầu ra. Cần model tải từ ngoài, Python tương thích và GPU để chạy nhanh; xem [README](../voice-clone-web/README.md).
- **Nói đúng giới hạn:** checkbox consent là bước nhắc trách nhiệm, không tự xác minh quyền sở hữu giọng. API chưa có xác thực cho triển khai công khai. [Bộ ôn chi tiết](voice-clone-web-interview/voice-clone-web/00-MUC-LUC.md).

### Word Checker - rà soát tài liệu Word

- **Giá trị:** kiểm tra định dạng, lỗi cơ học văn bản, chính tả, tiêu đề và giới hạn trong `.docx`; có CLI/web, báo cáo và bản sửa.
- **Quyết định kỹ thuật:** [runner](../word-checker/checker/runner.py) ghép các checker, [formatting](../word-checker/checker/formatting.py) xử lý font/style, [Issue](../word-checker/checker/issues.py) chuẩn hóa kết quả; [CLI](../word-checker/cli.py) và [web](../word-checker/webapp.py) dùng chung lõi.
- **Demo:** dùng file `.docx` mẫu không chứa thông tin cá nhân, chạy kiểm tra, mở báo cáo rồi thử một bản sửa **sao chép**. Cách chạy ở [README](../word-checker/README.md).
- **Nói đúng giới hạn:** heuristic/từ điển có thể bỏ sót hoặc báo nhầm; kiểm tra AI là tùy chọn và cần dịch vụ ngoài. Định dạng `.doc` cũ không phải đầu vào OOXML trực tiếp. [Bộ ôn chi tiết](word-checker-interview/word-checker/00-MUC-LUC.md).

### ContractShield & Redline Studio - kiểm toán rủi ro hợp đồng & quản trị vòng đời
- **Giá trị:** phát hiện 20 nhóm điều khoản bẫy song ngữ Việt - Anh (kèm Custom Playbook), sửa câu chữ 1-Click, mô phỏng tiền thiệt hại What-If, trích xuất lịch nhắc hẹn `.ics`, hỏi-đáp tình huống trên hợp đồng, quét ảnh hợp đồng giấy bằng OCR offline (`vie+eng`), soạn email đàm phán 3 giọng văn, che/khôi phục PII và niêm phong SHA-256 chống tráo bản in ký.
- **Quyết định kỹ thuật:** tự xây dựng bộ đọc/ghi `.docx` (ZIP + CRC32 + OOXML) và giải nén luồng `FlateDecode`/`UTF-16BE` cho `.pdf` thuần Node.js trong [server.js](../contract-intelligence/server.js) để không phụ thuộc gói cài đặt dễ gãy; quy hoạch động LCS word-diff kết hợp bắt thay đổi ngữ nghĩa (`phải` -> `có thể`, đảo số tiền); tách các engine mô phỏng tài chính, lịch RFC 5545 `.ics`, Q&A hybrid (offline retrieval + tùy chọn Gemini API) và OCR offline trong [v3-engines.js](../contract-intelligence/v3-engines.js); kiểm thử đầu-cuối qua [test-e2e.js](../contract-intelligence/test-e2e.js).
- **Demo:** chọn một mẫu hợp đồng có gài bẫy (lao động, phần mềm hoặc MSA tiếng Anh), bấm 1-Click sửa bẫy lên 100/100, kéo thanh trượt mô phỏng tiền phạt cứu được, thử giả lập tráo 1 chữ số để còi SHA-256 bắt sống, và chạy `npm test`. Xem [README](../contract-intelligence/README.md).
- **Nói đúng giới hạn:** tập quy tắc regex/heuristic và tính toán What-If là công cụ hỗ trợ rà soát kỹ thuật, **không thay thế tư vấn pháp lý chính thức của luật sư hành nghề**; bộ đọc PDF thuần Node.js chỉ bóc tách lớp văn bản số `FlateDecode` (bản PDF scan dạng ảnh cần chuyển sang tab Quét Ảnh OCR); Q&A mặc định dùng truy xuất điều khoản cục bộ khi không nhập Gemini API key.

## 4. Mẫu trả lời vòng kỹ thuật

**Kể một dự án trong 2 phút:** 20 giây nêu người dùng/vấn đề -> 30 giây cho luồng chính -> 40 giây cho quyết định khó và phương án đã chọn -> 20 giây cho bằng chứng (test/demo) -> 10 giây cho giới hạn và bước tiếp theo. Ví dụ với OTP Vault: vấn đề là lưu TOTP an toàn trên máy cá nhân; luồng là secret -> HMAC tạo mã, mật khẩu -> scrypt -> khóa Fernet mã hóa kho; bằng chứng là test vector RFC và test API; giới hạn là mất mật khẩu thì mất khả năng giải mã, app chỉ chạy local.

**Câu hỏi thường gặp và hướng trả lời:**

| Câu hỏi | Trả lời dựa trên bằng chứng |
|---|---|
| "Bạn trực tiếp làm phần nào?" | Chỉ rõ file/commit/tính năng bạn thực hiện; tách phần thư viện, model/API bên ngoài và đóng góp của đồng đội hoặc AI hỗ trợ. |
| "Vì sao chọn vanilla JS?" | Nêu bài toán cụ thể: app nhỏ, PWA tĩnh, không cần backend/build ở nhiều dự án; thừa nhận chi phí quản lý state khi app lớn. |
| "Có bao nhiêu người dùng, tác động thế nào?" | Chỉ đưa số từ analytics/triển khai thật. Nếu chưa đo, nói "chưa có số người dùng đáng tin cậy" và dùng demo/test làm bằng chứng kỹ thuật. |
| "Bạn đã kiểm thử gì?" | Nêu đúng test của dự án được hỏi; không lấy số test Quán Quen để đại diện cho cả portfolio. |
| "Nếu đưa vào production?" | Chọn rủi ro cụ thể: auth/TTL cho Agent Talk, review y tế cho Cấp Cứu 101, auth và quản lý consent cho Voice Clone, hardening token store cho Outlook. |
| "AI làm hộ bạn bao nhiêu?" | Trình bày quy trình thực tế: ý tưởng, thiết kế, kiểm tra output, sửa lỗi và test; chỉ nhận phần bạn có thể giải thích tại code. |

## 5. Chuẩn bị demo và bằng chứng

1. Chốt **2 dự án theo JD**; mỗi demo tối đa 3-5 phút, bắt đầu từ nhu cầu người dùng rồi mới mở code. Chuẩn bị một luồng ngắn đã tập chạy và phương án dự phòng bằng ảnh/video nếu phụ thuộc API, tài khoản hoặc GPU.
2. Kiểm tra môi trường theo README riêng; static ES modules cần HTTP server. Các app phụ thuộc dịch vụ ngoài gồm Agent Talk (LLM key), Outlook (Azure/tài khoản), Voice Clone (model/GPU), InkToWord (OCR qua mạng) và Art Shield (model tải ngoài).
3. Dùng dữ liệu giả: secret TOTP giả, mail thử nghiệm, log mẫu, tài liệu mẫu, giọng có quyền sử dụng. Ẩn `.env`, API key, refresh token, mail cá nhân và nội dung log trước khi chia sẻ màn hình.
4. Chuẩn bị một sơ đồ luồng và mở sẵn 2-3 file code cho mỗi dự án chính. Có thể dùng [bản đồ tài liệu từng dự án](README.md), [mẫu STAR](KyNang_PhongVan/02_CAU_CHUYEN_STAR.md) và [hướng dẫn viết CV](KyNang_PhongVan/04_VIET_CV_DU_AN.md) để tập giải thích quyết định, lỗi gặp phải và kết quả.
5. Trước khi ghi CV, kiểm tra lại số liệu test, ảnh chụp, link demo, mức hoàn thiện và đóng góp cá nhân. Không ghi "production", "bảo mật tuyệt đối", "AI tự phát triển" hoặc "hiệu quả được chứng minh" nếu không có bằng chứng phù hợp.

## 6. Kiểm tra đã thực hiện khi soạn hồ sơ

- Đối chiếu danh sách 12 thư mục ứng dụng, README, entrypoint và bộ ôn riêng tương ứng.
- `totp-manager`: `pytest -q` trong `.venv` hiện có -> **92 passed**.
- `red-team-lab`: `npm test` -> **3 suite đạt** (18 + 10 + 23 kiểm tra theo output).
- `quan-quen-map`: `npm run check` -> **đạt** kiểm tra encoding, cú pháp JS và manifest; repo có 47 khai báo test Playwright, nhưng lượt này **chưa chạy** Playwright.
- Các test/demo khác **chưa chạy trong lượt soạn tài liệu này**; hãy chạy lại trên máy sẽ dùng khi phỏng vấn.
