# 2. Các khái niệm nền tảng (lý thuyết)

File này giải thích **mọi thuật ngữ** trong dự án, viết cho người chưa biết gì. Nắm chắc phần này sẽ giúp bạn tự tin trước mọi câu hỏi "khái niệm".

---

## 2.1. PWA (Progressive Web App)

**PWA là gì?** Là một web app được "nâng cấp" để hành xử gần như app native: cài được lên màn hình chính, có icon riêng, chạy toàn màn hình (không thanh địa chỉ), và **chạy offline**. Người dùng không cần vào App Store/CH Play.

**Ba thành phần làm nên một PWA:**
1. **HTTPS** (hoặc localhost) — bắt buộc vì lý do bảo mật.
2. **Web App Manifest** (`manifest.json`) — file JSON khai báo tên app, icon, màu, chế độ hiển thị, shortcut.
3. **Service Worker** — một script chạy nền, chặn request mạng để phục vụ từ cache → offline.

**Trong dự án:** PWA cực kỳ quan trọng vì đây là app cấp cứu — **Chế độ Hoảng loạn phải dùng được kể cả khi mất mạng**. Manifest còn khai báo 2 shortcut: mở thẳng "Chế độ Hoảng loạn" (`?panic=1`) và "Công cụ khẩn cấp" (`?tools=1`).

---

## 2.2. Service Worker & Cache API

**Service Worker (SW) là gì?** Một file JavaScript chạy ở "tầng nền", **tách biệt khỏi trang web**, đóng vai trò như một **proxy mạng phía client**. Nó có thể chặn (intercept) mọi request mạng của trang và quyết định: trả từ cache hay đi mạng.

**Vòng đời của SW (lifecycle):**
- **install**: cài đặt — đây là lúc precache (tải sẵn) các file vào Cache API.
- **activate**: kích hoạt — dọn cache cũ.
- **fetch**: mỗi khi trang xin một tài nguyên, SW chặn lại để xử lý.

**Cache API:** kho lưu trữ request/response ngay trong trình duyệt, do SW quản lý. Khác với localStorage (chỉ lưu chuỗi text nhỏ), Cache API lưu được cả file (HTML/CSS/JS/ảnh).

**Hai chiến lược cache dùng trong dự án:**
- **Network-first** (cho điều hướng trang): thử mạng trước, rớt mạng thì trả trang từ cache. Đảm bảo người dùng thấy bản mới nhất khi có mạng.
- **Stale-while-revalidate** (cho các tài nguyên còn lại): trả ngay bản cache (nhanh), đồng thời âm thầm tải bản mới về cache cho lần sau. Cân bằng giữa tốc độ và độ mới.

**Cache versioning:** dự án dùng hằng `CACHE_VERSION = "capcuu101-v17"`. Mỗi lần đổi file phải **bump (tăng)** số version → SW mới sẽ dọn cache cũ và nạp bản mới. Đây là kỹ thuật quan trọng để tránh người dùng kẹt ở bản cũ.

> 💡 **Câu hỏi hay gặp: "Vì sao SW không chạy với `file://`?"** Vì lý do bảo mật, SW chỉ hoạt động trên `https://` hoặc `localhost`. Mở file trực tiếp bằng `file://` sẽ không đăng ký được SW.

---

## 2.3. Web Audio API

**Là gì?** Một API mạnh để tạo và xử lý âm thanh ngay trong trình duyệt theo thời gian thực, bằng cách nối các "node" âm thanh thành một đồ thị (graph).

**Cách dự án dùng (rất đáng nói):** Thay vì tải các file `.mp3`/`.wav` (tốn dung lượng, phải cache), dự án **tự tổng hợp (synthesize) toàn bộ âm thanh từ con số 0**:
- `AudioContext` — "động cơ" âm thanh.
- `OscillatorNode` — tạo sóng âm (square/triangle/sawtooth) ở một tần số → ra một "nốt".
- `GainNode` — điều khiển âm lượng theo thời gian (tạo hiệu ứng tăng/giảm).
- Ghép nhiều nốt theo thời gian → tiếng "tách" metronome, "ting" đúng, "kèn tụt dốc" thua, fanfare thắng, và cả nhạc nền 8-bit.

➡️ **Lợi ích:** không file media → app shell siêu nhẹ, **chạy offline hoàn hảo**, không cần cache file âm thanh.

**Lưu ý quan trọng — autoplay policy:** trình duyệt **cấm phát âm thanh** trước khi người dùng tương tác (chạm/click). Vì vậy dự án có hàm `unlock()` gọi `AudioContext` trong lần chạm đầu tiên để "mở khoá" audio.

---

## 2.4. Web Speech API (Text-to-Speech)

**Là gì?** API cho phép trình duyệt **đọc to văn bản** (SpeechSynthesis) — biến chữ thành giọng nói.

**Trong dự án:** dùng cho Chế độ Hoảng loạn — đọc to từng bước sơ cứu để người dùng rảnh tay làm theo.

**Các khái niệm cần biết:**
- `speechSynthesis.getVoices()` — lấy danh sách giọng có trên máy. **Bất đồng bộ**: lần đầu có thể trả mảng rỗng, phải lắng sự kiện `onvoiceschanged`.
- `SpeechSynthesisUtterance` — một "câu nói" với text, ngôn ngữ (`lang`), tốc độ, cao độ.
- **Vấn đề thực tế:** giọng phụ thuộc **hệ điều hành**. Nếu máy không có giọng `vi-VN`, trình duyệt sẽ lấy giọng tiếng Anh đọc chữ Việt → nghe sai hoàn toàn. Dự án xử lý bằng: trình chọn giọng trong Cài đặt, **cảnh báo** khi thiếu giọng Việt, và tùy chọn **tắt giọng đọc** (đọc theo chữ trên màn hình).

> 💡 **Mẹo TTS thú vị trong dự án:** Số khẩn cấp được viết là "một một năm" trong dữ liệu nguồn để TTS đọc đúng "115" (nếu viết "115" máy có thể đọc "một trăm mười lăm"). Khi **hiển thị** trên màn hình thì code thay lại thành "115". Tương tự "năm centimet" → "5cm". Đây là chi tiết tinh tế cho thấy bạn hiểu rõ hành vi của TTS.

---

## 2.5. Các Web API trình duyệt khác

- **Vibration API** (`navigator.vibrate`): làm thiết bị rung theo một mẫu (pattern). Dùng để rung theo nhịp ép tim. Chủ yếu hoạt động trên Android.
- **DeviceOrientation API**: đọc cảm biến con quay (gyroscope) để biết thiết bị đang nghiêng thế nào. Dùng cho game Heimlich "nghiêng điện thoại đẩy bụng". iOS yêu cầu xin quyền; máy không có cảm biến thì fallback sang chạm.
- **Notifications API**: hiển thị thông báo hệ thống. Dùng để nhắc ôn tập. *Lưu ý:* dự án chỉ nhắc **khi mở lại app** (không có push nền) vì push nền cần server + VAPID — chưa làm.
- **Web Share API** (`navigator.share`): gọi hộp thoại chia sẻ native của thiết bị. Dùng để chia sẻ thẻ kết quả. Không hỗ trợ thì fallback chép link.
- **Canvas API**: vẽ đồ hoạ 2D bằng code. Dùng để **vẽ thẻ ảnh kết quả** (điểm/sao/huy hiệu) rồi chia sẻ.
- **localStorage**: kho key-value trong trình duyệt, lưu chuỗi text, còn sau khi đóng tab.

---

## 2.6. i18n (Internationalization — đa ngôn ngữ)

**i18n là gì?** Là việc thiết kế app để hỗ trợ nhiều ngôn ngữ. (Viết tắt vì giữa "i" và "n" của "internationalization" có 18 chữ.)

**Cơ chế i18n tự viết trong dự án (không thư viện):**
- Biến `LANG` đọc từ `localStorage.capcuu101_lang` (mặc định `vi`).
- Hàm `t(v)`: nếu `v` là object `{vi, en}` → trả về theo `LANG`; nếu là chuỗi → trả nguyên.
- **Chuỗi giao diện tĩnh**: gom trong object `UI` (key → `{vi, en}`); phần tử HTML gắn `data-i18n="key"`; hàm `applyLang()` quét tất cả và đặt `textContent`.
- **Nội dung động**: dữ liệu song ngữ ngay trong cấu trúc (tên màn, bài học là object `{vi,en}`; quiz/panic có bản song song `QUIZZES_EN`/`PANIC_EN`).
- **Khối văn xuôi dài trong HTML**: dùng class `.only-vi`/`.only-en` để ẩn/hiện theo `body.lang-en`.

> 💡 Đây là một ví dụ rất hay để nói về việc **tự xây một hệ thống i18n đơn giản mà đủ dùng**, hiểu được trade-off giữa "tự viết" và "dùng thư viện".

---

## 2.7. IIFE & module pattern

**IIFE (Immediately Invoked Function Expression)** là một hàm được định nghĩa và **gọi ngay lập tức**: `(() => { ... })()`.

**Vì sao dùng?** Không có hệ thống module (vì không build), nên bọc toàn bộ `app.js` trong một IIFE để:
- **Tạo scope riêng**: các biến/hàm bên trong không "rò rỉ" ra global → tránh xung đột tên.
- Chỉ chủ động lộ ra cái cần thiết (ví dụ `sound.js` lộ `window.Sfx`).
- `"use strict"` để bật chế độ nghiêm ngặt, bắt lỗi sớm.

---

## 2.8. Một số thuật ngữ khác

- **App shell**: bộ khung tối thiểu (HTML/CSS/JS) cần để app hiển thị; được precache để chạy offline.
- **Router (phía client)**: bộ điều hướng đổi "màn hình" mà không tải lại trang. Ở đây là hàm `go(id)` đổi class `.active` giữa các `<section class="screen">`.
- **Splash screen**: màn hình chào khi mở app.
- **Accordion**: thành phần UI gập/mở từng mục (dùng cho Cẩm nang).
- **ARIA (Accessible Rich Internet Applications)**: tập thuộc tính HTML giúp trình đọc màn hình hiểu giao diện (vd `aria-label`, `aria-live`, `aria-hidden`).
- **`prefers-reduced-motion`**: tín hiệu từ hệ điều hành cho biết người dùng muốn giảm hiệu ứng chuyển động; dự án tự tôn trọng tín hiệu này.
- **XSS (Cross-Site Scripting)**: lỗ hổng khi chèn mã độc qua nội dung người dùng nhập. Dự án chống bằng cách dùng `textContent` (escape) khi hiển thị dữ liệu thẻ y tế.
- **Open Graph / Twitter Card**: các thẻ `<meta>` giúp link khi dán lên mạng xã hội hiển thị đẹp (ảnh + tiêu đề + mô tả).
- **Metronome**: bộ đếm nhịp; ở đây phát nhịp 100-110/phút để ép tim đúng tốc độ.
- **Streak (chuỗi ngày)**: số ngày liên tiếp người dùng mở app ôn tập.
