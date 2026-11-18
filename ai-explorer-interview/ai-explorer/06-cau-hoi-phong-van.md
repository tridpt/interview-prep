# 6. Ngân hàng câu hỏi phỏng vấn & trả lời mẫu

40+ câu, chia theo nhóm, từ dễ đến khó. Mỗi câu có **gợi ý trả lời**. Đọc rồi tự diễn đạt lại bằng lời của bạn.

> ⭐ = câu rất hay được hỏi. Ưu tiên luyện kỹ.

---

## A. Tổng quan & động lực (warm-up)

**A1. ⭐ Giới thiệu ngắn gọn về dự án.**
> AI Explorer là "bảo tàng AI tương tác" gồm 26 phòng chạy hoàn toàn trên trình duyệt. Mỗi phòng giải thích trực quan một khái niệm AI và cho tự tay nghịch — từ webcam dạy AI, mạng nơ-ron, token, embeddings, attention, tới RAG, agents, RLHF và cả chain-of-thought. 100% vanilla JS + ES Modules, không backend, không API key, là PWA chạy offline, song ngữ Việt/Anh.

**A2. Vì sao bạn làm dự án này?**
> Em muốn giúp người mới *hiểu AI bằng trực giác* thay vì đọc công thức, và tự thử thách xây mọi thứ bằng vanilla JS: router, i18n, PWA, deep-link, cả pipeline test. Mỗi phòng là một trò chơi nhỏ minh hoạ một khái niệm.

**A3. Đối tượng người dùng?**
> Hai nhóm: người mới hoàn toàn (lộ trình *Người mới*, 5 phòng cốt lõi) và lập trình viên muốn nắm khái niệm để xây app AI (lộ trình *Cho lập trình viên*: token, embeddings, RAG, fine-tuning, agents, reasoning).

**A4. Vì sao gọi là "hiểu AI trong 15 phút"?**
> Vì thiết kế ưu tiên trực giác: không toán, không thuật ngữ rối; mỗi khái niệm là một demo bấm-là-thấy. Lộ trình *Người mới* đi hết trong ~10 phút.

---

## B. Kiến trúc & thiết kế

**B1. ⭐ Mô tả kiến trúc tổng thể.**
> SPA phía client, không backend. `app.js` ghép metadata `ROOM_META` với `ROOM_LOADERS` thành `ROOMS` (mạch kể chuyện), nạp module phòng khi cần; router bằng `hashchange`, toolbar và các hệ con (i18n, store, sound, roomstate, roomquiz). `pwa.js`+`sw.js` lo offline. Dữ liệu ở localStorage.

**B2. ⭐ Vì sao không dùng backend / API key?**
> Không cần — mọi phòng tự tính tại chỗ bằng JS (KNN, MLP, k-means, bigram, mô phỏng). Không backend giúp: chạy offline, riêng tư tuyệt đối, miễn phí (không tốn phí API), deploy đơn giản (web tĩnh GitHub Pages), không server để sập.

**B3. ⭐ Vì sao chọn vanilla JS + ES Modules, không framework?**
> Mục tiêu là app giáo dục nhẹ, offline-first, không phụ thuộc. Vanilla + ES Modules cho app shell nhỏ để cache, không bề mặt tấn công chuỗi cung ứng, không build phức tạp, mỗi phòng là module cô lập. Đổi lại em tự viết router/i18n/PWA — cũng là phần học được nhiều nhất.

**B4. ES Modules là gì và vì sao phải chạy qua HTTP server?**
> Là hệ module chuẩn của JS (`import`/`export`), mỗi file một module. Trình duyệt nạp module qua `fetch`, mà `fetch` bị chặn với `file://` (CORS), nên phải chạy qua `http://localhost`, ví dụ `python -m http.server`.

**B5. Mảng `ROOMS` đóng vai trò gì?**
> Là nguồn sự thật duy nhất cho điều hướng: thứ tự = mạch kể chuyện. Router, thanh tiến trình, nút Trước/Sau, lưới trang chủ, ô tìm kiếm, và cả smoke test đều dựa vào nó. Thêm/bớt phòng chỉ sửa một chỗ.

**B6. `route()` hoạt động thế nào?**
> Khi hash đổi: phát `roomleave` → xoá DOM → `applyTheme` → nếu home thì render home; ngược lại tìm phòng trong `ROOMS`, đánh dấu đã ghé, dựng khung sườn, `await room.render()` để nạp phòng, kiểm tra route còn hợp lệ rồi gọi hàm render trả về, chèn quiz và nút điều hướng.

**B7. Trạng thái app lưu ở đâu?**
> localStorage dưới khoá `ai-explorer-progress`: phòng đã ghé, điểm quiz cao nhất, câu nhỏ đã đúng, lộ trình. Ngôn ngữ và cài đặt âm thanh lưu ở khoá riêng. Mọi đọc/ghi bọc try/catch cho Safari private mode.

**B8. Router dùng hash hay History API? Vì sao?**
> Dùng hash (`#/tokenizer`) + sự kiện `hashchange`. Hash hoạt động ngay trên hosting tĩnh (GitHub Pages) không cần cấu hình server rewrite, và là nơi tự nhiên để gắn deep-link state.

---

## C. PWA & Offline (trọng tâm)

**C1. ⭐ PWA là gì? Cần những gì?**
> Web app cài được lên màn hình chính và chạy offline như app native. Cần: HTTPS (hoặc localhost), Web App Manifest, và Service Worker.

**C2. ⭐ Service worker trong dự án hoạt động thế nào?**
> install: precache mảng `ASSETS` vào Cache API. activate: xoá cache cũ khác `CACHE` rồi `clients.claim()`. fetch: cache-first cho asset nội bộ, đi mạng cho phần còn lại, fallback `index.html` khi offline.

**C3. ⭐ Cache versioning là gì? Vì sao quan trọng?**
> Hằng `CACHE = "ai-explorer-v22"`. Mỗi lần đổi file phải bump version; khi SW mới activate nó xoá mọi cache tên khác → người dùng không kẹt bản cũ. Đây là kỹ thuật cốt lõi để phân phối bản cập nhật với SW.

**C4. ⭐ Làm sao người dùng nhận bản mới mà không kẹt bản cũ?**
> SW mới cố tình KHÔNG tự `skipWaiting` — nó "chờ" để không đổi asset giữa chừng. `pwa.js` hiện toast "Đã có bản mới — Tải lại"; bấm nút gửi `postMessage("SKIP_WAITING")`; worker mới `skipWaiting()` rồi chiếm quyền (`controllerchange`) → trang tự reload một lần.

**C5. Vì sao service worker không chạy với `file://`?**
> Lý do bảo mật — SW chỉ chạy trên `https://` hoặc `localhost`. Mở file trực tiếp không đăng ký được SW.

**C6. Chiến lược cache của bạn là gì, vì sao chọn vậy?**
> Cache-first cho asset cùng origin (nhanh, offline tốt vì asset gần như tĩnh giữa các version), đi mạng cho tài nguyên ngoài (vd Google Fonts), và fallback `index.html` khi offline. Vì đổi bản là bump version nên cache-first không gây kẹt.

**C7. Offline quan trọng thế nào với dự án này?**
> Rất — app học nên dùng được cả trên máy bay, mạng yếu, hay demo không mạng. Vì mọi phòng tính tại chỗ, offline gần như "miễn phí" khi đã cache app shell.

---

## D. i18n & song ngữ

**D1. ⭐ Cơ chế đa ngôn ngữ của bạn?**
> Biến `lang` từ localStorage; `tx({vi,en})` trả chuỗi theo ngôn ngữ, hoặc `tx(a,b)` chọn nhanh. Nội dung song ngữ lưu ngay trong cấu trúc dữ liệu (mỗi phòng title/question/blurb là {vi,en}). Đổi ngôn ngữ phát `langchange` → dựng lại toolbar + render lại phòng.

**D2. Vì sao tự viết i18n thay vì dùng thư viện?**
> Nhu cầu đơn giản (2 ngôn ngữ), tự viết chỉ ~20 dòng, tránh thêm phụ thuộc đúng triết lý no-deps. Thư viện i18n đầy đủ sẽ thừa và làm nặng app shell.

**D3. Làm sao đảm bảo không quên dịch một chuỗi?**
> Có test `i18n.mjs` quét mọi file `.js`, dựng lại object literal để tìm mọi `{vi,en}` và báo lỗi nếu thiếu một vế hoặc để chuỗi rỗng. Nó duyệt từng ký tự (bỏ qua string/comment) thay vì regex vì chuỗi nằm trong template literal lồng nhau.

**D4. Đổi ngôn ngữ giữa chừng thì phòng đang mở có cập nhật không?**
> Có — `setLang` phát `langchange`; `app.js` bắt sự kiện, dựng lại toolbar và gọi `route()` để render lại phòng hiện tại, nên mọi chuỗi động đổi theo ngay.

---

## E. JavaScript / vanilla & quản lý tài nguyên

**E1. ⭐ Không có framework, bạn quản lý vòng đời phòng thế nào?**
> `route()` phát sự kiện `roomleave` trước khi đổi phòng. Mỗi phòng lắng `roomleave` (once) để dọn timer/animation/webcam. Không tự dọn sẽ rò rỉ — vd metronome/interval chạy mãi hay camera vẫn bật.

**E2. Ví dụ một rò rỉ tài nguyên bạn phải xử lý?**
> Phòng webcam: khi rời phòng phải `stream.getTracks().forEach(t => t.stop())` để tắt camera; phòng có animation phải `clearInterval`/`clearTimeout`. Reasoning giữ mảng `timers` và clear khi `roomleave` hoặc khi chọn bài mới.

**E3. Bạn chống XSS/nội dung không mong muốn thế nào?**
> Nội dung do dev kiểm soát (không nhận HTML tuỳ ý từ người dùng). Deep-link đọc tham số rồi ép kiểu (vd `parseInt(getParam("p"))` và kiểm tra hợp lệ) nên không nhét được HTML. Nếu mở rộng nhận input người dùng, em sẽ dùng `textContent` thay `innerHTML`.

**E4. Vì sao dùng `history.replaceState` cho deep-link?**
> Để cập nhật URL "âm thầm" khi người dùng gõ/chọn mà KHÔNG phát `hashchange` (tránh render lại phòng và tránh làm ngập lịch sử back). `replaceState` thay URL hiện tại tại chỗ.

**E5. Âm thanh được tạo ra sao? Xử lý autoplay policy thế nào?**
> Tự tổng hợp bằng Web Audio: oscillator + gain, không file media. Trình duyệt cấm phát âm trước tương tác, nên `AudioContext` chỉ được tạo/`resume()` khi cần (sau khi người dùng bấm).

---

## F. Khái niệm AI (các phòng)

**F1. ⭐ Phòng "Tự tay dạy AI" minh hoạ gì?**
> Học từ ví dụ (KNN): AI học từ dữ liệu mẫu bạn tự chụp qua webcam, không phải luật viết tay. Nó rút ảnh về lưới 24×24 xám rồi phân loại theo hàng xóm gần nhất. Bài học: dữ liệu là "nhiên liệu", dữ liệu lệch thì AI lệch.

**F2. Mạng nơ-ron trong phòng 02 thực sự học thế nào?**
> Là MLP 2→H→1 tự cài `forward` + lan truyền ngược thủ công trong `trainStep`. Kéo số nơ-ron: 1 nơ-ron chỉ vẽ được đường thẳng nên không tách nổi vòng tròn; nhiều nơ-ron ghép nhiều đường thẳng thành đường cong — minh hoạ "deep learning".

**F3. Giải thích overfitting bằng ngôn ngữ đời thường.**
> Là khi AI "học vẹt" — nhớ vanh vách dữ liệu train nhưng sai trên dữ liệu mới (test). Cách chữa: nhiều dữ liệu hơn, mô hình đơn giản hơn, hoặc regularization.

**F4. Token là gì? Vì sao tính chi phí theo token?**
> AI không đọc chữ như người — nó cắt văn bản thành token (mảnh subword). Model xử lý và tính chi phí theo số token, không theo số chữ cái, nên câu dài/ngôn ngữ hiếm tốn nhiều token hơn.

**F5. Embeddings và ví dụ "vua − đàn ông + đàn bà = nữ hoàng" nghĩa là gì?**
> Embeddings biến từ thành vector (điểm trên bản đồ nghĩa); từ gần nghĩa thì gần nhau, và *khoảng cách/hướng* mang nghĩa quan hệ — nên phép cộng trừ vector ra được loại suy.

**F6. Attention là gì?**
> Cơ chế cho mỗi từ "nhìn" vào các từ khác để hiểu ngữ cảnh (vd "nó" trỏ về đâu). Đây là nền tảng của Transformer và của các LLM hiện đại.

**F7. Vì sao AI "ảo giác" (hallucination)?**
> Vì nó sinh chữ bằng cách đoán từ tiếp theo theo xác suất, không kiểm chứng sự thật. Nó chọn từ *nghe hợp lý*, nên đôi khi bịa mà vẫn rất tự tin.

**F8. RAG là gì, giải quyết vấn đề gì?**
> Retrieval-Augmented Generation: trước khi trả lời, AI *tra* đoạn tài liệu liên quan rồi đưa vào ngữ cảnh, nên trả lời bám tài liệu riêng, bớt bịa và có dẫn nguồn — mà không phải huấn luyện lại model.

**F9. Prompting và fine-tuning khác nhau chỗ nào?**
> Prompting: chỉ dẫn model ngay lúc dùng (không đổi trọng số) — rẻ, nhanh, linh hoạt. Fine-tuning: huấn luyện lại trên dữ liệu riêng để đổi trọng số — tốn kém hơn nhưng "ngấm" phong cách/kiến thức. Prompt/RAG trước, fine-tune khi thật cần.

**F10. AI agent khác chatbot thường thế nào?**
> Agent không chỉ trả lời — nó lên kế hoạch, gọi công cụ (máy tính, tìm kiếm…), đọc kết quả rồi lặp nhiều bước tới khi hoàn thành nhiệm vụ.

**F11. Cửa sổ ngữ cảnh (context window) là gì?**
> Lượng token tối đa model "nhớ" cùng lúc. Hội thoại quá dài thì token cũ rơi ra khỏi cửa sổ → AI "quên" phần đầu.

**F12. Prompt injection là gì, vì sao lừa được AI?**
> Là nhét "câu lệnh lén" vào nội dung để AI làm trái chỉ dẫn gốc. Nó lừa được vì AI đọc lời hệ thống và lời người dùng chung một dòng chữ, khó tách bạch đâu là "lệnh thật".

**F13. RLHF là gì?**
> Reinforcement Learning from Human Feedback: con người xếp hạng các câu trả lời → huấn luyện một "mô hình phần thưởng" theo sở thích đó → tinh chỉnh model để trả lời hữu ích, lịch sự hơn. Đây là bước "dạy AI cư xử".

---

## G. Chain-of-thought & reasoning models (phòng mới — trọng tâm)

**G1. ⭐ Chain-of-thought (CoT) là gì?**
> Là cho model *viết ra các bước suy luận trung gian* trước khi đưa đáp án cuối, thay vì trả lời ngay. Ví dụ đếm chữ "r" trong "strawberry": trả lời ngay dễ ra "2" (sai); đánh vần → đánh dấu → đếm thì ra "3" (đúng).

**G2. ⭐ Vì sao cho model "nghĩ" từng bước lại chính xác hơn?**
> Vì model sinh chữ bằng cách đoán từ tiếp theo. Trả lời ngay chỉ là "một phát đoán". Khi viết từng bước, mỗi bước trở thành ngữ cảnh cho bước sau, nên lời giải được *dựng dần* thay vì đoán một lần. Model không thông minh hơn — nó có thêm *không gian để tính*.

**G3. ⭐ CoT khác reasoning models (o1, DeepSeek-R1) ở đâu?**
> CoT là *kỹ thuật prompt* ("hãy nghĩ từng bước"). Reasoning model là *mô hình được huấn luyện* để tự động sinh một chuỗi suy nghĩ dài (thường ẩn) trước khi chốt — không cần nhắc. Nói cách khác, reasoning model nội tại hoá CoT.

**G4. ⭐ Đánh đổi khi cho model "nghĩ" là gì?**
> Chậm hơn và tốn nhiều token hơn (chi phí + độ trễ). Phòng minh hoạ bằng huy hiệu token: trả lời ngay ≈1–3 token, nghĩ từng bước ≈48–70 token. Vì vậy chỉ nên "nghĩ kỹ" cho bài khó.

**G5. Khi nào nên dùng reasoning / CoT, khi nào không?**
> Nên: toán, logic, lập trình nhiều bước, bài cần suy luận nhiều chặng. Không cần: câu đơn giản, tra cứu nhanh, tác vụ nhạy độ trễ/chi phí — vì "nghĩ" thừa vừa chậm vừa tốn token.

**G6. Bạn cài phòng Reasoning thế nào (kỹ thuật)?**
> Deterministic: mỗi bài có sẵn đáp án bốc đồng (sai) + chuỗi bước + đáp án đúng + số token minh hoạ. "Trả lời ngay" hiện đáp án đỏ tức thì; "Suy nghĩ từng bước" hiện lần lượt các bước bằng `setTimeout` (tôn trọng `prefers-reduced-motion` — nếu bật thì hiện ngay), rồi đáp án xanh. Bài chọn được deep-link qua tham số `p`, và có quiz nhỏ. Timer được dọn khi rời phòng/đổi bài.

---

## H. Testing & CI/CD

**H1. ⭐ Dự án test thế nào?**
> Ba lớp, đều chạy trong CI trước deploy: (1) `check-assets.mjs` bảo đảm mọi asset runtime nằm trong `ASSETS` của `sw.js`; (2) `i18n.mjs` bảo đảm mọi `{vi,en}` đủ hai vế; (3) `smoke.mjs` dùng Playwright duyệt mọi phòng và fail nếu có lỗi console/exception. Ngoài ra có `focus-trap.mjs` và `pwa.mjs`.

**H2. ⭐ `check-assets` giải quyết vấn đề gì?**
> Chống "offline hỏng thầm lặng": nếu thêm file mới mà quên đưa vào `ASSETS`, offline sẽ thiếu file mà không báo lỗi lúc dev. Test này đọc `ASSETS` từ `sw.js`, so với mọi `.js`/core/icon thực tế, và fail nếu thiếu — chặn deploy.

**H3. Vì sao smoke test đọc `ROOMS` từ mã nguồn?**
> Để phòng mới thêm sau này *tự động* được kiểm tra, không phải sửa file test. Test trích `id` từ mảng `ROOMS`, duyệt từng phòng bằng Chromium, và bắt mọi lỗi console/exception/request hỏng.

**H4. Deploy thế nào?**
> GitHub Actions: job `test` chạy trước, chỉ khi pass mới tới job `deploy` đẩy lên GitHub Pages. Sau khi đổi asset nhớ bump `CACHE` để client nhận bản mới.

**H5. Vì sao viết bộ duyệt ký tự cho `i18n.mjs` thay vì regex?**
> Vì chuỗi song ngữ nằm trong template literal nhiều dòng, mảng, và có nháy lồng nhau — regex sẽ cho kết quả sai. Bộ duyệt ký tự có stack biết chính xác mỗi key `vi`/`en` thuộc object nào.

---

## I. Accessibility

**I1. ⭐ Bạn làm gì cho accessibility?**
> Tôn trọng `prefers-reduced-motion` (tắt độ trễ animation, vd ở Reasoning); ARIA cho toolbar/tiến trình/ô tìm; bẫy focus cho hộp thoại tìm; điều hướng bàn phím (←/→, `/`, ↑↓/Enter/Esc); và tự tính tương phản màu WCAG cho chữ.

**I2. Giải thích cách tự tính tương phản màu.**
> `luminance()` tính độ sáng tương đối theo WCAG; `contrast()` tính tỉ lệ; `darkenForText()` làm đậm dần màu chủ đề tới khi tỉ lệ với nền sáng ≥ 4.5:1, gán vào `--accent-ink` chỉ dùng cho chữ. Khối màu giữ nguyên độ rực.

**I3. Bẫy focus là gì và bạn cài ra sao?**
> Giữ Tab/Shift+Tab quẩn trong dialog (ô tìm là phần tử focus duy nhất), và trả focus về nút đã mở dialog khi đóng (lưu `document.activeElement` trước, `.focus()` lại khi đóng). Kèm `role="dialog"`, `aria-modal`.

---

## J. Mở rộng & tư duy

**J1. ⭐ Thêm một phòng mới cần làm gì?**
> Tạo `rooms/x.js` export `render(root)`; thêm import + mục vào `ROOMS`; thêm `THEME`, (tuỳ chọn) `HINTS`, `SEARCH_KEYWORDS`, quiz trong `BANK`; thêm file vào `ASSETS` của `sw.js` và bump `CACHE`; (tuỳ chọn) thêm vào lộ trình. Khung sườn tự lo; smoke test tự kiểm tra.

**J2. ⭐ Thử thách kỹ thuật khó nhất?**
> (Chọn một để kể sâu) Tự tính tương phản WCAG cho màu từng phòng; hoặc bẫy focus + ARIA cho ô tìm; hoặc luồng cập nhật PWA "không kẹt bản cũ"; hoặc bộ duyệt ký tự cho test i18n. Kể theo vấn đề → cách giải → kết quả.

**J3. Trade-off lớn nhất của dự án?**
> No-framework/no-build cho nhẹ, offline tốt, no-deps — đổi lại tự viết router/i18n/PWA và tự quản vòng đời phòng (dễ rò rỉ nếu quên dọn). Em bù bằng quy ước `roomleave` + test tự động.

**J4. Nếu có thêm thời gian bạn cải thiện gì?**
> Thêm test đơn vị cho từng thuật toán phòng; thêm nhiều ngôn ngữ (i18n đã sẵn cấu trúc); tách CSS theo phòng; và có thể thêm phòng mới theo xu hướng (vd MoE, tool-calling nâng cao).

**J5. Phòng bạn tự hào nhất và vì sao?**
> Phòng Reasoning — vừa cập nhật xu hướng (reasoning models o1/DeepSeek-R1) vừa gói gọn nhiều kỹ thuật: so sánh trả lời ngay vs từng bước, huy hiệu token thể hiện đánh đổi, deep-link, a11y, quiz. Nó chứng minh kiến trúc module hoá thật sự dễ mở rộng.

**J6. Dự án "vỡ" ở đâu trên môi trường thiếu tính năng?**
> Máy không cho quyền webcam → phòng 01 hiện thông báo và vẫn đọc được giải thích; không có Web Share → fallback copy link; không có Web Audio → âm thanh im lặng (không lỗi). Em thiết kế fallback nhẹ nhàng ở hầu hết chỗ.

---

## 🎤 5 câu hay gặp nhất — luyện trước tiên
1. A1 — Giới thiệu dự án.
2. B3 — Vì sao vanilla JS + ES Modules, không framework.
3. C2/C4 — Service worker & luồng cập nhật không kẹt bản cũ.
4. G1/G2 — Chain-of-thought là gì, vì sao "nghĩ" giúp đúng hơn.
5. H1/H2 — Ba lớp test & vì sao `check-assets` quan trọng.

## ❓ Câu hỏi NÊN HỎI LẠI nhà tuyển dụng
- Team có làm PWA/offline-first không, dùng chiến lược cache gì?
- Quy trình test & chặn deploy (CI) của team ra sao?
- Team ưu tiên accessibility tới mức nào trong sản phẩm?
