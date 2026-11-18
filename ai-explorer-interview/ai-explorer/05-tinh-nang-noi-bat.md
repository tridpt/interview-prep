# 5. Tính năng nổi bật (phần "ăn điểm")

Đây là các tính năng nên chủ động kể để gây ấn tượng. Mỗi mục gồm: **vấn đề → giải pháp → vì sao ấn tượng**.

---

## 5.1. Chạy offline hoàn toàn, không backend, không API key

- **Vấn đề**: app học AI thường phụ thuộc server/API — mất mạng là hỏng, và cần key (tốn tiền, lộ khoá).
- **Giải pháp**: mọi phòng **tự tính tại chỗ** bằng JS/canvas (KNN, MLP, k-means, bigram, mô phỏng diffusion…), service worker cache toàn bộ app shell → chạy trọn vẹn offline.
- **Ấn tượng**: "Không một dòng gọi mạng, không API key — mọi 'AI' trong app là thuật toán thật chạy trên máy người dùng, nên vừa riêng tư, vừa miễn phí, vừa offline."

---

## 5.2. Kiến trúc module hoá dễ mở rộng

- **Vấn đề**: 26 phòng dễ biến `app.js` thành "món spaghetti".
- **Giải pháp**: mỗi phòng là một module `.js` cô lập export `render(root)`; `app.js` chỉ điều phối qua mảng `ROOMS` (một nguồn sự thật duy nhất). Thêm phòng = tạo file + thêm 1 mục vào `ROOMS` + thêm vào `ASSETS`.
- **Ấn tượng**: "Khung sườn (tiêu đề, tiến trình, điều hướng, quiz) được xử lý tự động; phòng chỉ lo nội dung. Đây là separation of concerns mà không cần framework."

---

## 5.3. Song ngữ VN/EN toàn diện + test bảo vệ

- **Vấn đề**: dịch thiếu một chuỗi là lỗi thầm lặng, khó soi giữa hàng nghìn chuỗi.
- **Giải pháp**: `i18n.js` với `tx({vi,en})`; đổi ngôn ngữ phát `langchange` → render lại tức thì. Có test `i18n.mjs` **quét từng ký tự** mọi object `{vi,en}` để bắt vế thiếu hoặc chuỗi rỗng.
- **Ấn tượng**: "Em không dựa vào regex đơn giản (dễ sai với template literal lồng nhau) mà viết một bộ duyệt ký tự có stack để biết mỗi key `vi`/`en` thuộc object nào — rồi đưa vào CI."

---

## 5.4. PWA có thông báo bản mới (không kẹt bản cũ)

- **Vấn đề**: service worker cache mạnh tay dễ khiến người dùng **kẹt ở bản cũ** mãi.
- **Giải pháp**: SW mới cố tình **không tự `skipWaiting`** mà "chờ"; `pwa.js` hiện toast **"Đã có bản mới — Tải lại"**; bấm nút mới gửi `SKIP_WAITING`, worker mới chiếm quyền và trang tự reload.
- **Ấn tượng**: "Em ưu tiên trải nghiệm ổn định — không đổi asset giữa chừng — nhưng vẫn cho người dùng chủ động cập nhật bằng một cú bấm. Kèm cache versioning (`ai-explorer-v22`) để dọn cache cũ sạch sẽ."

---

## 5.5. Deep-link chia sẻ đúng một trạng thái

- **Vấn đề**: chia sẻ "phòng Tokenizer" thì được, nhưng chia sẻ *đúng câu mình đang gõ* thì không.
- **Giải pháp**: `roomstate.js` mã hoá trạng thái vào query của hash (`#/reasoning?p=2`), dùng `replaceState` để cập nhật âm thầm; nút Chia sẻ mang theo URL đó (Web Share API/clipboard).
- **Ấn tượng**: "URL trở thành trạng thái chia sẻ được, mà vẫn hoạt động trên hosting tĩnh (GitHub Pages) không cần server rewrite."

---

## 5.6. Tìm nhanh phòng có bẫy focus (accessibility)

- **Vấn đề**: 26 phòng thì cần tra nhanh; modal mà không quản focus là lỗi a11y kinh điển.
- **Giải pháp**: phím `/` mở ô tìm; lọc theo tên + từ khoá (chuẩn hoá không dấu bằng `normalize("NFD")`); ↑↓/Enter/Esc; **bẫy focus** giữ Tab trong dialog và **trả focus về nút mở** khi đóng; đủ ARIA (`role="dialog"`, `aria-modal`, `role="listbox/option"`, `aria-activedescendant`).
- **Ấn tượng**: "Em xử lý focus như một component thư viện thực thụ, chỉ bằng vanilla JS."

---

## 5.7. Tự tính tương phản màu theo chuẩn WCAG

- **Vấn đề**: mỗi phòng một tông màu rực — chữ màu đó trên nền sáng dễ **không đạt tương phản** (khó đọc, fail WCAG).
- **Giải pháp**: `luminance()` + `contrast()` (đúng công thức WCAG) + `darkenForText()` làm đậm dần màu (×0.9/vòng) tới khi đạt tỉ lệ ≥ 4.5:1, gán vào biến `--accent-ink` dùng riêng cho chữ. Khối màu giữ nguyên độ rực.
- **Ấn tượng**: "Accessibility không phải 'thêm cho có' — em tính tương phản bằng code để mọi phòng đều đọc được, mà vẫn giữ được thẩm mỹ Neo-brutalism."

---

## 5.8. CI chặn deploy nếu test fail

- **Vấn đề**: dự án nhiều file dễ "quên" — quên cache một asset (offline hỏng thầm lặng), quên dịch, hay phòng mới gây lỗi console.
- **Giải pháp**: GitHub Actions (`deploy.yml`) chạy job `test` trước khi deploy, gồm:
  1. `check-assets.mjs` — mọi asset runtime phải có trong `ASSETS` của `sw.js`.
  2. `i18n.mjs` — mọi `{vi,en}` đủ hai vế.
  3. `seo.mjs` — kiểm tra trang tĩnh song ngữ/sitemap từ `build-seo.mjs`.
  4. `smoke.mjs` — Playwright duyệt **mọi phòng** (đọc `ROOMS` từ mã nguồn), fail nếu có lỗi console/exception.
  5. `focus-trap.mjs`, `pwa.mjs`, `a11y.mjs` — focus, PWA và accessibility.
  Job Lighthouse riêng kiểm tra ngân sách SEO/accessibility, theo dõi hiệu năng; deploy đợi cả hai job.
  Fail bất kỳ bước nào → **deploy bị chặn**.
- **Ấn tượng**: "`check-assets` là 'lưới an toàn' cho offline — nếu em thêm phòng mà quên cache file, CI báo lỗi ngay thay vì để offline hỏng âm thầm. Và smoke test tự đọc `ROOMS` nên không bao giờ 'quên test' phòng mới."

---

## 5.9. ⭐ Phòng Reasoning — chain-of-thought (điểm nhấn thời sự)

- **Vấn đề**: nhiều tài liệu AI chưa bắt kịp làn sóng *reasoning models* (o1, DeepSeek-R1).
- **Giải pháp**: phòng 25 cho người dùng chọn một bài "bẫy" (đếm chữ *r* trong *strawberry*, bài toán bút–tẩy kiểu CRT, tính thứ trong tuần), rồi so sánh trực tiếp:
  - **⚡ Trả lời ngay** → đáp án bốc đồng, thường **sai**, tô đỏ, kèm huy hiệu **≈1–3 token**.
  - **💭 Suy nghĩ từng bước** → hiện lần lượt từng bước suy luận (có độ trễ, tôn trọng `prefers-reduced-motion`), rồi ra đáp án **đúng**, tô xanh, huy hiệu **≈48–70 token**.
- **Ấn tượng**: "Phòng này dạy đúng ý tưởng cốt lõi của reasoning models: cho model *không gian để tính* — mỗi bước viết ra thành ngữ cảnh cho bước sau — và **huy hiệu token cho thấy đánh đổi tốc độ/chi phí ⟷ độ chính xác**. Nó deterministic, song ngữ, có deep-link (tham số `p`), có quiz nhỏ, và nằm trong lộ trình *Cho lập trình viên*. Đây là bằng chứng em theo sát xu hướng và biến nó thành trải nghiệm học được."

> 💡 **Mẹo phỏng vấn**: Dùng phòng Reasoning làm "câu chuyện đinh". Nó vừa thể hiện kỹ thuật (mô-đun mới, timer, deep-link, a11y) vừa thể hiện hiểu biết AI cập nhật (CoT, reasoning models, đánh đổi token).
