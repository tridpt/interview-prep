# 7. Cheat Sheet — Xem nhanh trước giờ G

Một trang tóm tắt. Đọc lướt 5-10 phút trước khi vào phỏng vấn.

---

## 🎯 Pitch 30 giây (học thuộc)
> Cấp Cứu 101: web app **PWA** dạy sơ cứu qua mini-game hài hước kiểu Dumb Ways to Die, kèm **Chế độ Hoảng loạn** đọc to từng bước khi gặp chuyện thật. 100% **HTML/CSS/JS thuần** — không framework, không build, không thư viện — **chạy offline 100%** nhờ service worker. Dùng nhiều Web API (Web Audio, Web Speech, Vibration, DeviceOrientation, Web Share), **song ngữ Việt/Anh**, dữ liệu lưu cục bộ (localStorage).

## 🧱 Tech stack
- **Cốt lõi**: Vanilla HTML/CSS/JS (ES6+), không build step.
- **PWA**: Web App Manifest + Service Worker + Cache API.
- **Web APIs**: Web Audio (tự synth âm thanh), Web Speech (TTS), Vibration, DeviceOrientation, Notifications, Web Share, Canvas, localStorage.
- **Dev tools**: Python + Pillow (gen_*.py sinh ảnh — không phải runtime).
- **Deploy**: GitHub Pages (web tĩnh).

## 🗂️ Các file
| File | Vai trò |
|---|---|
| `index.html` | 10 màn (section.screen) + overlay, meta PWA/OG |
| `sound.js` | window.Sfx — Web Audio synth (nạp TRƯỚC app.js) |
| `app.js` | Bộ não (IIFE ~2.980 dòng): i18n, router, dữ liệu, engine, Panic, công cụ |
| `style.css` | Theme, animation, a11y |
| `sw.js` | Service worker — offline |
| `manifest.json` | Khai báo PWA + 2 shortcut |

## 📐 Kiến trúc
Client-side only, **không backend**. Mọi logic + dữ liệu + state ở trình duyệt. localStorage lưu sao/stats/prefs/lang/voice/thẻ y tế.

## 🌊 Service Worker (nhớ kỹ)
- **install** → precache app shell (Promise.allSettled) → skipWaiting.
- **activate** → xoá cache cũ ≠ CACHE_VERSION → clients.claim.
- **fetch** → navigate: **network-first**; asset: **stale-while-revalidate**.
- Đổi asset → **bump CACHE_VERSION** (vd v17→v18).
- SW chỉ chạy qua `http(s)://`, không với `file://`.

## 🔊 Web Audio (tự synth)
Oscillator (sóng) + Gain (bao âm lượng) → tách/ting/thua/thắng/nhạc nền. Không file media → app nhẹ, offline tốt. **Autoplay policy**: phải `unlock()` audio trong user gesture đầu.

## 🗣️ TTS (Panic Mode)
getVoices() **async** (lắng onvoiceschanged); thiếu giọng vi-VN → giọng Anh đọc Việt sai → có **cảnh báo + chọn giọng + tắt giọng**. Mẹo "một một năm"→đọc "115", hiển thị thay lại "115".

## 🎮 Engine game
5 kiểu: rhythm / thrust / hose / press / quiz (17 màn). Mỗi game đặt **`gameState.cleanup`**; `stopGame()` gọi cleanup khi rời màn → chống rò rỉ tài nguyên. Survival: hỏi liên tục + đếm ngược (requestAnimationFrame).

## 🌐 i18n
LANG (localStorage) + `t()`; text tĩnh `data-i18n` + applyLang(); nội dung động `{vi,en}` / QUIZZES_EN, PANIC_EN; văn xuôi dài `.only-vi/.only-en`.

## 🧩 An toàn nội dung (app y tế)
**Một nguồn dữ liệu**: Cẩm nang + Panic dùng chung PANIC/PANIC_EN (DRY). Đối chiếu AHA/Mayo/NHS/WHO. `MEDICAL_REVIEW.md` cho nhân viên y tế. Disclaimer "không thay đào tạo, gọi 115". **Chế độ nghiêm túc** bỏ dark humor.

## 🔐 Bảo mật & riêng tư (3 điểm)
1. Mọi dữ liệu ở localStorage, không gửi đi đâu.
2. Thẻ y tế: escape bằng `textContent` (chống XSS); `tel:` lọc số `[\d+]`.
3. Không thư viện ngoài → không bề mặt tấn công chuỗi cung ứng.

## ♿ Accessibility
Tương phản cao, chữ lớn, giảm chuyển động (prefers-reduced-motion), chế độ nghiêm túc; ARIA (label/live/hidden), focus rõ, ÉP TIM bằng Space/Enter, quiz là button.

## ⚖️ Trade-off chính
No-framework/no-build → nhẹ, offline tốt, no-deps; đổi lại tự viết nhiều + app.js lớn khó mở rộng. Hài hước → dễ nhớ/lan toả; đổi lại cần chế độ nghiêm túc + disclaimer.

## 🚀 Cải thiện tương lai
Test tự động (kiểm toàn vẹn dữ liệu y tế); tách module; web push thật (server+VAPID); rà soát y tế/bản dịch bản ngữ; đánh giá WCAG với screen reader.

## ⚠️ Giới hạn đã biết (chủ động nêu)
Chưa có test tự động; TTS phụ thuộc giọng hệ thống; cảm biến/rung/tel chủ yếu trên điện thoại; thẻ y tế kẹt sau khoá màn hình; app.js file lớn.

## 💬 Thuật ngữ chốt
- **PWA / Service Worker / Cache API / app shell**
- **network-first / stale-while-revalidate / cache versioning**
- **Web Audio / oscillator / autoplay policy**
- **Web Speech (TTS) / getVoices async**
- **Vibration / DeviceOrientation / Web Share / Canvas**
- **i18n / IIFE / client-side routing**
- **localStorage / XSS / DRY / prefers-reduced-motion / ARIA**

---

### ✅ Checklist tâm lý trước phỏng vấn
- [ ] Pitch dự án trong 30 giây.
- [ ] Giải thích vòng đời service worker + 2 chiến lược cache.
- [ ] Lý do chọn vanilla JS (offline-first, no-deps).
- [ ] Cách xử lý TTS đa nền tảng.
- [ ] Nêu được tư duy an toàn nội dung cho app y tế.
- [ ] Nêu 2 giới hạn + cách khắc phục.
- [ ] Chuẩn bị 2 câu hỏi ngược lại nhà tuyển dụng.

**Hít thở sâu. Bạn hiểu dự án này. Cứ tự tin kể câu chuyện của mình. 🍀**
