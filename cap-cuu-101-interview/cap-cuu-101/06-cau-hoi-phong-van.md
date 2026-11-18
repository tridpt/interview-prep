# 6. Ngân hàng câu hỏi phỏng vấn & trả lời mẫu

60+ câu, chia theo nhóm, từ dễ đến khó. Mỗi câu có **gợi ý trả lời**. Đọc rồi tự diễn đạt lại bằng lời của bạn.

> ⭐ = câu rất hay được hỏi. Ưu tiên luyện kỹ.

---

## A. Tổng quan & động lực (warm-up)

**A1. ⭐ Giới thiệu ngắn gọn về dự án.**
> Cấp Cứu 101 là web app PWA dạy sơ cứu sinh tử qua mini-game hài hước kiểu Dumb Ways to Die, kèm Chế độ Hoảng loạn đọc to từng bước khi gặp chuyện thật. Toàn bộ là HTML/CSS/JS thuần — không framework, không build, không thư viện — và chạy offline 100% nhờ service worker. Dùng nhiều Web API (Web Audio, Web Speech, Vibration, DeviceOrientation, Web Share), song ngữ Việt/Anh, dữ liệu lưu cục bộ.

**A2. Vì sao bạn làm dự án này?**
> Em muốn làm thứ vừa hữu ích thật (giúp người ta nhớ kỹ năng cứu người) vừa thử thách kỹ thuật — đặc biệt là yêu cầu offline-first cho app cấp cứu, và làm mọi thứ bằng vanilla JS không phụ thuộc.

**A3. Đối tượng người dùng?**
> Người bình thường muốn học sơ cứu một cách nhẹ nhàng, người lớn tuổi (nhờ chữ lớn/tương phản cao), và lớp học/môi trường y tế (chế độ nghiêm túc).

**A4. ⭐ Vì sao gọi là "đùng để chết nhảm" / dùng phong cách hài hước cho app y tế?**
> Lấy cảm hứng từ Dumb Ways to Die — dùng sự hài hước, châm biếm để kiến thức ghi nhớ tự nhiên, vô thức. Nhưng em ý thức rủi ro nên có **Chế độ nghiêm túc** bỏ dark humor cho môi trường y tế, và disclaimer rõ ràng "không thay thế đào tạo chính quy, việc thật gọi 115".

---

## B. Kiến trúc & thiết kế

**B1. ⭐ Mô tả kiến trúc tổng thể.**
> App hoàn toàn phía client, không backend. `index.html` chứa 10 màn dạng section; `sound.js` lộ window.Sfx (âm thanh Web Audio); `app.js` là bộ não (i18n, router, dữ liệu, engine game, Panic, công cụ) bọc trong IIFE; `sw.js` là service worker cho offline. Mọi dữ liệu người dùng ở localStorage.

**B2. ⭐ Vì sao không dùng backend?**
> Không cần — nội dung tĩnh, mọi logic chạy được ở client. Không backend giúp: chạy offline, riêng tư tuyệt đối (dữ liệu y tế cá nhân không gửi đi đâu), deploy đơn giản (web tĩnh GitHub Pages), không server để sập hay bảo trì.

**B3. ⭐ Vì sao chọn vanilla JS, không framework?**
> Yêu cầu lớn nhất là offline-first và nhẹ tuyệt đối. Mỗi phụ thuộc là một rủi ro (kích thước, bảo mật chuỗi cung ứng, build phức tạp). Vanilla cho toàn quyền kiểm soát và app shell nhỏ để cache. Đổi lại em tự viết router/i18n/state — cũng là phần học được nhiều nhất.

**B4. Nhược điểm của việc dồn logic vào một file app.js lớn?**
> File ~2.980 dòng khó bảo trì khi mở rộng. Em đã ghi nhận đây là giới hạn; hướng cải thiện là tách module (ES modules) khi quy mô lớn hơn. Hiện giữ một file để không cần build step.

**B5. Router của bạn hoạt động thế nào?**
> Hàm `go(id)`: dừng hoạt động màn cũ (game/panic/metronome để tránh rò rỉ), đổi class `.active` giữa các section, cập nhật `aria-hidden`, rồi gọi hàm render tương ứng. Đây là client-side routing đơn giản không tải lại trang.

**B6. Trạng thái app lưu ở đâu?**
> Toàn bộ ở localStorage: sao, thống kê, cài đặt âm thanh, tùy chọn accessibility, ngôn ngữ, giọng đọc, thẻ y tế. Mỗi loại một khóa riêng có tiền tố `capcuu101_`.

---

## C. PWA & Offline (trọng tâm)

**C1. ⭐ PWA là gì? Cần những gì để thành PWA?**
> Web app cài được lên màn hình chính và chạy offline như app native. Cần: HTTPS (hoặc localhost), Web App Manifest, và Service Worker.

**C2. ⭐ Service worker hoạt động thế nào trong dự án?**
> Lúc install: precache toàn bộ app shell vào Cache API. Lúc activate: xoá cache cũ không thuộc version hiện tại. Lúc fetch: chặn request — điều hướng trang dùng network-first, asset dùng stale-while-revalidate.

**C3. ⭐ Giải thích 2 chiến lược cache và vì sao dùng khác nhau.**
> Network-first cho trang HTML: thử mạng trước (để có bản mới), rớt mạng thì trả từ cache → không kẹt bản cũ khi có mạng. Stale-while-revalidate cho asset: trả cache ngay (nhanh), cập nhật ngầm cho lần sau → asset ít đổi nên ưu tiên tốc độ.

**C4. ⭐ Làm sao người dùng nhận được bản cập nhật?**
> Bump `CACHE_VERSION` mỗi lần đổi asset. SW mới khi activate sẽ xoá cache cũ; `skipWaiting()` + `clients.claim()` để áp dụng ngay không cần đóng app.

**C5. Vì sao service worker không chạy với `file://`?**
> Lý do bảo mật — SW chỉ chạy trên `https://` hoặc `localhost`. Mở file trực tiếp không đăng ký được SW, nên phải chạy qua HTTP (vd `python -m http.server`).

**C6. `Promise.allSettled` khi precache để làm gì?**
> Để một file lỗi (vd thiếu) không làm hỏng toàn bộ quá trình cài SW — các file khác vẫn được cache. An toàn hơn `Promise.all` (fail nếu một cái lỗi).

**C7. Offline quan trọng thế nào với dự án này?**
> Sống còn — đây là app cấp cứu, lúc khẩn cấp có thể mất mạng. Chế độ Hoảng loạn phải dùng được offline. Đó là lý do em ưu tiên offline-first ngay từ đầu.

---

## D. Web APIs

**D1. ⭐ Vì sao tự tổng hợp âm thanh bằng Web Audio thay vì dùng file?**
> Để app shell siêu nhẹ và offline hoàn hảo — không phải cache file media. Em ghép oscillator (sóng) + gain (bao âm lượng) tạo mọi hiệu ứng và nhạc nền 8-bit từ con số 0.

**D2. ⭐ Autoplay policy là gì? Bạn xử lý sao?**
> Trình duyệt cấm phát âm thanh trước khi người dùng tương tác. Em có hàm `unlock()` gọi/`resume()` AudioContext trong lần chạm đầu tiên để mở khoá audio.

**D3. ⭐ TTS (Web Speech) gặp vấn đề gì và bạn giải quyết thế nào?**
> getVoices() bất đồng bộ (phải lắng onvoiceschanged); giọng phụ thuộc hệ điều hành — thiếu giọng vi-VN thì giọng Anh đọc chữ Việt sai. Em xử lý: trình chọn giọng, cảnh báo khi thiếu giọng Việt, tùy chọn tắt giọng đọc.

**D4. Mẹo "một một năm" → "115" để làm gì?**
> Để TTS đọc đúng "một-một-năm" thay vì "một trăm mười lăm". Trong dữ liệu nguồn viết "một một năm"; khi hiển thị thì code thay lại thành "115". Tương tự "năm centimet" → "5cm".

**D5. DeviceOrientation dùng ở đâu? Fallback thế nào?**
> Game Heimlich "nghiêng điện thoại đẩy bụng" dùng cảm biến con quay. iOS phải xin quyền; máy không có cảm biến thì fallback sang nhấn-giữ-thả.

**D6. Notifications trong dự án có hạn chế gì?**
> Chỉ nhắc khi mở lại app (không có push nền). Push nền kể cả khi đóng app cần hạ tầng web push (server + VAPID) — chưa làm vì dự án không có backend.

**D7. Web Share API hoạt động ra sao? Có fallback không?**
> Gọi hộp thoại chia sẻ native (`navigator.share`). Em vẽ thẻ ảnh kết quả bằng Canvas rồi chia sẻ; thiết bị không hỗ trợ thì fallback chép link vào clipboard.

---

## E. i18n

**E1. ⭐ Cơ chế đa ngôn ngữ của bạn?**
> Biến LANG từ localStorage; hàm t() trả chuỗi theo ngôn ngữ nếu dữ liệu là object {vi,en}. Text tĩnh dùng data-i18n + applyLang() quét đặt textContent; nội dung động lưu song ngữ trong cấu trúc; văn xuôi dài dùng class .only-vi/.only-en.

**E2. Vì sao tự viết i18n thay vì dùng thư viện?**
> Quy mô đủ nhỏ, nhu cầu đơn giản; tự viết tránh thêm phụ thuộc (đúng triết lý no-deps của dự án). Thư viện sẽ thừa và làm nặng app shell.

**E3. Thêm một chuỗi mới cần làm gì?**
> Thêm cùng key vào CẢ vi và en trong UI; HTML dùng data-i18n, JS dùng t(UI.key). Nội dung động phải cập nhật cả hai phiên bản (vd QUIZZES và QUIZZES_EN).

---

## F. Engine game & dữ liệu

**F1. ⭐ Có mấy kiểu game? Chúng tái sử dụng khung chung thế nào?**
> 5 kiểu: rhythm (nhịp), thrust (đẩy/nghiêng), hose (vòi nước), press (ép cầm máu), quiz (trắc nghiệm). 17 màn chỉ dùng 5 cơ chế này. Chung 3 phần tử DOM và biến gameState; runGame() điều phối theo `type`.

**F2. ⭐ `gameState.cleanup` là gì và vì sao quan trọng?**
> Mỗi game khi khởi tạo đặt một hàm cleanup (xoá listener, clear interval, dừng cảm biến). stopGame() gọi nó khi rời màn. Không có framework lo việc này nên phải tự dọn để chống rò rỉ tài nguyên — vd metronome chạy mãi hay cảm biến vẫn nghe.

**F3. Mô hình dữ liệu một màn?**
> LEVELS[] có {id, emoji, type, char, title{}, desc{}, lesson{}}. Quiz lưu trong QUIZZES[id] = mảng câu hỏi, mỗi câu options:[{t,ok}] đúng 1 đáp án. Panic lưu PANIC[id] = {title, metronome?, steps[]}.

**F4. Chế độ Survival khác gì game thường?**
> Hỏi liên tục câu rút ngẫu nhiên từ mọi tình huống, mỗi câu có đồng hồ đếm ngược (~9s, dùng requestAnimationFrame), sai/hết giờ là kết thúc, lưu kỷ lục best.survival.

**F5. Vì sao Cẩm nang và Panic dùng chung dữ liệu?**
> Nguyên tắc DRY và an toàn nội dung y tế: một nguồn dữ liệu, sửa một chỗ đúng mọi nơi, không bao giờ lệch nhau giữa hai màn.

---

## G. Bảo mật & riêng tư

**G1. ⭐ Dữ liệu người dùng có an toàn không?**
> Mọi dữ liệu (sao, thống kê, thẻ y tế) chỉ ở localStorage trên máy, không gửi đi đâu. Không backend nên không có rủi ro lộ dữ liệu phía server.

**G2. ⭐ Chống XSS ở đâu?**
> Thẻ y tế nhận input người dùng → render bằng `textContent` (escape) thay vì innerHTML để chèn mã không chạy được. Số điện thoại được lọc còn `[\d+]` trước khi tạo `tel:`.

**G3. Vì sao "không thư viện ngoài" cũng là một điểm bảo mật?**
> Không nhúng thư viện/CDN nghĩa là không có bề mặt tấn công chuỗi cung ứng phía client (supply-chain) — không lo một dependency bị chèn mã độc.

**G4. Thẻ y tế có hạn chế gì?**
> Nằm trong app — nếu khoá màn hình thì người cứu hộ không mở được. Nên dùng kèm Medical ID của hệ điều hành. Em ghi nhận đây là giới hạn.

---

## H. Accessibility & nội dung y tế

**H1. ⭐ Bạn làm gì cho accessibility?**
> Tương phản cao, chữ lớn, giảm chuyển động (tôn trọng prefers-reduced-motion), chế độ nghiêm túc; ARIA cho nút icon, aria-live đọc bước Panic, aria-hidden cho màn ẩn, focus rõ, nút ÉP TIM chơi bằng Space/Enter, quiz là button.

**H2. Game cảm ứng (kéo/nghiêng) có tiếp cận được bằng bàn phím không?**
> Khó — em thừa nhận điều này. Nhưng phần trắc nghiệm và Panic Mode là kênh học tập tiếp cận đầy đủ cho mọi người, kể cả dùng bàn phím/screen reader.

**H3. ⭐ Làm app y tế bạn lo gì nhất? Xử lý sao?**
> Nội dung sai là rủi ro lớn nhất. Em đối chiếu hướng dẫn AHA/Mayo Clinic/NHS/WHO/Red Cross, có file MEDICAL_REVIEW.md làm checklist cho nhân viên y tế rà soát, disclaimer rõ ở nhiều nơi, và một nguồn dữ liệu duy nhất để không lệch nội dung.

**H4. Chế độ nghiêm túc giải quyết vấn đề gì?**
> Phong cách hài hước "đăng xuất khỏi cuộc đời" không phù hợp lớp học/y tế. Chế độ nghiêm túc bỏ dark humor, dùng từ trung tính ("Hoàn thành đúng cách"/"Chưa đạt").

---

## I. Testing & vận hành

**I1. ⭐ Dự án test thế nào?**
> Thành thật: chưa có test tự động — đây là giới hạn đã biết. Hiện kiểm thử thủ công theo checklist (HUONG_DAN_TEST.md) + kiểm cú pháp bằng `node --check` cho JS và `python -m json.tool` validate manifest.

**I2. Nếu thêm test, bạn test gì trước?**
> Kiểm tra toàn vẹn dữ liệu — thứ nguy hiểm nhất với app y tế: mỗi quiz đúng 1 đáp án, mọi quiz có cả VI+EN, mọi panic-card có dữ liệu tương ứng. Đây là loại lỗi nội dung dễ lọt và nguy hiểm.

**I3. Deploy thế nào?**
> GitHub Pages — web tĩnh nên deploy = đẩy file. Sau khi đổi asset nhớ bump CACHE_VERSION để client nhận bản mới.

**I4. Khi dev bị "kẹt" bản cũ do service worker thì sao?**
> Do stale-while-revalidate. Cách xử lý: Ctrl+F5 hoặc Unregister service worker trong DevTools, và nhớ bump CACHE_VERSION.

---

## J. Câu hỏi mở / tư duy

**J1. ⭐ Thử thách khó nhất khi làm dự án?**
> (Gợi ý) Xử lý TTS đa nền tảng (getVoices async, thiếu giọng Việt, đọc số đúng); hoặc đảm bảo offline thật sự hoạt động với cache versioning. Kể vấn đề → cách giải → kết quả.

**J2. ⭐ Nếu có thêm thời gian bạn cải thiện gì?**
> Thêm test tự động (nhất là kiểm toàn vẹn dữ liệu y tế); tách app.js thành module; web push thật (server + VAPID) cho nhắc nền; nhờ nhân viên y tế VN + người bản ngữ rà soát nội dung/bản dịch; đánh giá WCAG đầy đủ với screen reader.

**J3. App "vỡ" ở đâu trên thiết bị yếu/thiếu tính năng?**
> Máy không có giọng vi-VN → TTS sai (đã cảnh báo + cho tắt); không có gyroscope → game nghiêng fallback chạm; không hỗ trợ Web Share → chép link; rung chủ yếu Android. Em thiết kế fallback cho hầu hết.

**J4. Vì sao dùng Python ở đây nếu là app JS thuần?**
> Python (Pillow) chỉ dùng lúc dev để sinh ảnh icon/screenshot/Open Graph (gen_*.py), chạy một lần, không nằm trong runtime của app.

**J5. Quyết định kỹ thuật bạn tự hào nhất?**
> Kiến trúc offline-first hoàn chỉnh bằng vanilla — service worker + tự tổng hợp âm thanh (không file media) → app shell nhỏ, chạy offline 100% cho một app cấp cứu. Mọi mảnh ghép phục vụ một mục tiêu rõ ràng.

**J6. Trade-off lớn nhất của dự án?**
> No-framework/no-build (nhẹ, offline tốt, không phụ thuộc) đổi lại tự viết nhiều và một file app.js lớn khó mở rộng. Và phong cách hài hước (dễ nhớ, lan toả) đổi lại phải có chế độ nghiêm túc + disclaimer cho môi trường y tế.

---

## K. Câu hỏi "code cụ thể" (có thể bị soi)

**K1. Thứ tự nạp sound.js và app.js có quan trọng không?**
> Có — sound.js nạp trước để lộ window.Sfx, app.js dùng nó. Đổi thứ tự sẽ lỗi "Sfx is undefined".

**K2. Vì sao bọc app.js trong IIFE?**
> Tạo scope riêng (biến không rò ra global, tránh xung đột), bật "use strict". Vì không có module system (không build) nên IIFE là cách cô lập code đơn giản.

**K3. `?panic=1` và `?tools=1` để làm gì?**
> Là tham số query cho 2 shortcut PWA — mở app vào thẳng Chế độ Hoảng loạn / Công cụ khẩn cấp. Quan trọng cho tình huống khẩn cấp, không phải bấm nhiều bước.

**K4. Khi đổi ngôn ngữ giữa chừng, màn đang mở có cập nhật không?**
> Có — setLang() lưu localStorage, gọi applyLang() rồi render lại màn đang mở để nội dung động (quiz/panic...) đổi theo.

**K5. Vì sao overlay kết quả gắn vào #screen-game chứ không phải #game-stage?**
> Để phủ toàn màn và cuộn được; nếu gắn vào stage sẽ bị giới hạn trong khung game.

---

## 🎤 5 câu hay gặp nhất — luyện trước tiên
1. A1 — Giới thiệu dự án.
2. C2/C3 — Service worker & 2 chiến lược cache.
3. B3 — Vì sao vanilla JS không framework.
4. D3 — Xử lý TTS đa nền tảng.
5. H3 — Làm app y tế lo gì, xử lý sao.

## ❓ Câu hỏi NÊN HỎI LẠI nhà tuyển dụng
- Team có làm PWA/offline-first không, dùng chiến lược cache gì?
- Quy trình kiểm thử & đảm bảo chất lượng nội dung (nhất là domain nhạy cảm) thế nào?
- Team ưu tiên accessibility tới mức nào trong sản phẩm?
