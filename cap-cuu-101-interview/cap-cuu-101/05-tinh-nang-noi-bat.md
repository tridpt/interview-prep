# 5. Phân tích sâu các tính năng "ăn điểm"

File này giúp bạn **kể chuyện ấn tượng**. Mỗi tính năng đi kèm: ý tưởng, cách làm, và "vì sao nó khó/hay".

---

## 5.1. Offline-first qua Service Worker ⭐ (quan trọng nhất)

**Ý tưởng:** một app cấp cứu **phải dùng được khi mất mạng** — đúng lúc người ta cần nhất.

**Cách làm:** service worker precache toàn bộ app shell (mọi file cục bộ) lúc `install`; sau đó mọi request được phục vụ từ cache. Dùng 2 chiến lược: network-first cho trang, stale-while-revalidate cho asset. Cache được version-hoá (`CACHE_VERSION`) để cập nhật sạch sẽ.

**Vì sao hay:** thể hiện tư duy **offline-first** và hiểu sâu vòng đời service worker (install/activate/fetch), Cache API, và bài toán cập nhật cache. Đây là kiến thức rất "thực chiến" cho web hiện đại.

**Có thể bị hỏi vặn:**
- *"Làm sao người dùng nhận bản mới?"* → Bump `CACHE_VERSION`; `activate` xoá cache cũ; `skipWaiting`+`clients.claim` để áp dụng ngay.
- *"Tại sao mất mạng vẫn vào được?"* → App shell đã nằm trong cache; `navigate` rớt mạng thì trả `index.html` từ cache.

---

## 5.2. Âm thanh tự tổng hợp bằng Web Audio ⭐

**Ý tưởng:** không dùng một file âm thanh nào — **tạo mọi tiếng động từ con số 0** bằng oscillator + gain.

**Cách làm:** ghép các `OscillatorNode` (sóng square/triangle/sawtooth) với `GainNode` (bao âm lượng theo thời gian) để tạo tiếng tách metronome, ting đúng, kèn thua, fanfare thắng, và nhạc nền 8-bit.

**Vì sao hay:**
- App shell **siêu nhẹ**, không phải cache file media → offline hoàn hảo.
- Thể hiện hiểu biết về **xử lý âm thanh số** (tần số, sóng, envelope) và **autoplay policy** (phải mở khoá audio trong user gesture đầu tiên).

---

## 5.3. Chế độ Hoảng loạn + TTS ⭐ (tính năng "linh hồn")

**Ý tưởng:** lúc khẩn cấp người ta hoảng, run tay, không đọc nổi chữ → app **đọc to từng bước bình tĩnh** để rảnh tay làm theo.

**Cách làm:** dùng Web Speech API (SpeechSynthesis) đọc từng bước; điều hướng Bước tiếp/Lùi/Đọc lại; riêng CPR có metronome 100/phút + rung dẫn nhịp ép tim; nút gọi 115 luôn hiện; nhạc nền tự tắt.

**Chi tiết tinh tế đáng kể:**
- Xử lý `getVoices()` bất đồng bộ (lắng `onvoiceschanged`).
- **Cảnh báo + trình chọn giọng** khi máy thiếu giọng `vi-VN` (nếu không, giọng Anh đọc chữ Việt sai hoàn toàn).
- Mẹo "một một năm" → đọc đúng "115", hiển thị thay lại "115".
- Đa ngôn ngữ: chế độ English đổi giọng `en-US` và nói "call emergency services" thay vì 115 (số khẩn cấp khác theo quốc gia).

**Vì sao hay:** đây là nơi **giá trị nhân văn gặp kỹ thuật** — bạn nghĩ tới trải nghiệm thật của người dùng trong cơn hoảng loạn, và xử lý hàng loạt ca biên của TTS.

---

## 5.4. Engine 5 kiểu game với mẫu `cleanup()` ⭐

**Ý tưởng:** 17 màn chỉ dùng 5 cơ chế game (`rhythm/thrust/hose/press/quiz`), tái sử dụng chung khung.

**Điểm kỹ thuật đáng nói — quản lý vòng đời thủ công:** không có framework lo việc dọn dẹp, nên mỗi game đặt `gameState.cleanup` (xoá listener, clear interval, dừng cảm biến); `stopGame()` gọi nó khi rời màn. Nếu quên, sẽ **rò rỉ tài nguyên** (vd metronome vẫn chạy, cảm biến vẫn nghe).

**Vì sao hay:** cho thấy bạn hiểu **quản lý vòng đời và bộ nhớ** — thứ framework thường làm hộ, nhưng ở đây bạn tự làm.

---

## 5.5. i18n song ngữ tự viết ⭐

**Ý tưởng:** hỗ trợ đầy đủ Việt/Anh — giao diện, nội dung game, quiz, panic, giọng đọc — mà không thư viện.

**Cách làm:** từ điển `UI` + hàm `t()` + thuộc tính `data-i18n` cho text tĩnh; dữ liệu song ngữ `{vi,en}` cho nội dung động; class `.only-vi/.only-en` cho văn xuôi dài.

**Vì sao hay:** thể hiện khả năng **tự thiết kế một hệ thống nhỏ gọn đủ dùng** và hiểu trade-off tự-viết vs thư viện.

---

## 5.6. Một nguồn dữ liệu (DRY) cho nội dung y tế ⭐

**Ý tưởng:** Chế độ Hoảng loạn và Cẩm nang tra cứu dùng **chung** dữ liệu `PANIC`/`PANIC_EN`.

**Vì sao hay (đặc biệt quan trọng):** với app y tế, **nội dung sai là rủi ro nguy hiểm nhất**. Một nguồn dữ liệu duy nhất nghĩa là sửa một chỗ thì đúng mọi nơi — không bao giờ có tình trạng Cẩm nang nói một kiểu, Panic nói kiểu khác. Đây là quyết định thiết kế có ý thức về **an toàn nội dung**.

---

## 5.7. Khả năng tiếp cận (Accessibility) & Chế độ nghiêm túc ⭐

**Ý tưởng:** app phải dùng được cho người lớn tuổi, người khiếm thị, và phù hợp môi trường lớp học/y tế.

**Cách làm:**
- **Tương phản cao**, **chữ lớn**, **giảm chuyển động** (tự tôn trọng `prefers-reduced-motion` của hệ điều hành).
- **Chế độ nghiêm túc**: bỏ phần "đăng xuất khỏi cuộc đời", dùng từ trung tính → hợp lớp học/y tế.
- ARIA: `aria-label` cho nút icon, `aria-live` đọc bước Panic & hướng dẫn game, `aria-hidden` cho màn ẩn, focus rõ, nút ÉP TIM chơi bằng Space/Enter, quiz là `<button>`.

**Vì sao hay:** accessibility là điểm thường bị bỏ quên; nêu được nó chứng tỏ bạn nghĩ cho **mọi loại người dùng**, không chỉ trường hợp lý tưởng.

---

## 5.8. Công cụ khẩn cấp dùng thật

- **Máy đếm nhịp CPR độc lập**: bấm một nút phát nhịp 110/phút (tách + rung + đếm 1-30) — dùng ngay khi ép tim thật.
- **Gọi khẩn cấp 1 chạm**: 115/113/114/111 qua `tel:`.
- **Thẻ y tế cá nhân**: nhóm máu, dị ứng, bệnh nền, người liên hệ — lưu offline, escape chống XSS, nút gọi người thân. Hiển thị thẻ to rõ cho người cứu hộ.

**Vì sao hay:** biến app từ "trò chơi" thành **công cụ cứu hộ thật** — cho thấy bạn nghĩ trọn vẹn về vòng đời sử dụng (học → khi cần → dùng).

---

## 5.9. Chia sẻ bằng Canvas + Web Share API

**Ý tưởng:** sau khi thắng, tạo một tấm thẻ ảnh đẹp (điểm/sao/huy hiệu) để khoe.

**Cách làm:** **vẽ thẻ bằng Canvas** (đồ hoạ 2D), rồi gọi **Web Share API** gửi qua mạng xã hội/tin nhắn; không hỗ trợ thì chép link. Kèm Open Graph/Twitter Card để link preview đẹp.

**Vì sao hay:** kết hợp 2 API trình duyệt (Canvas vẽ + Web Share gửi) cho một cơ chế lan toả tự nhiên, có fallback.

---

## 5.10. Bảng tổng hợp "tính năng ↔ khái niệm kỹ thuật" (học thuộc)

| Tính năng | Khái niệm kỹ thuật bạn thể hiện |
|-----------|-------------------------------|
| Offline-first | Service Worker (lifecycle), Cache API, 2 chiến lược cache, cache versioning |
| Âm thanh tự tổng hợp | Web Audio (oscillator/gain), autoplay policy |
| Panic Mode + TTS | Web Speech API, xử lý async getVoices, ca biên giọng nói |
| Engine game | quản lý vòng đời thủ công, chống rò rỉ (cleanup) |
| i18n | tự thiết kế hệ thống đa ngôn ngữ |
| Một nguồn dữ liệu | DRY, an toàn nội dung |
| Accessibility | ARIA, prefers-reduced-motion, thiết kế bao trùm |
| Công cụ khẩn cấp | tel:, Vibration, thiết kế hướng tới tình huống thật |
| Chia sẻ | Canvas, Web Share API, Open Graph, fallback |
| Cảm biến nghiêng | DeviceOrientation, xin quyền iOS, fallback |

> 💡 **Chiến lược kể chuyện:** nếu được hỏi "tính năng nào em tự hào nhất?", chọn **offline-first** (chiều sâu kỹ thuật) hoặc **Chế độ Hoảng loạn + TTS** (giá trị nhân văn + nhiều ca biên). Cả hai đều cho thấy bạn nghĩ cho người dùng thật.
