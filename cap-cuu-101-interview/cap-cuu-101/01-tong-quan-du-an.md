# 1. Tổng quan dự án Cấp Cứu 101

## 1.1. Cấp Cứu 101 là gì?

Cấp Cứu 101 (tên đầy đủ: *Đừng Để Chết Nhảm*) là một **web app dạy sơ cứu** qua mini-game hài hước, lấy cảm hứng từ *Dumb Ways to Die*. Ý tưởng cốt lõi: **học kỹ năng sinh tử bằng cách chơi vui**, để kiến thức "ghi nhớ một cách vô thức", rồi khi gặp chuyện thật thì có công cụ dùng ngay.

Hãy hình dung: bạn chơi mini-game "ép tim cho Gấu Béo" theo đúng nhịp 100-120/phút, làm sai thì nhân vật "đăng xuất khỏi cuộc đời" rất hài. Nhưng khi sự cố thật xảy ra, bạn mở **Chế độ Hoảng loạn**, chọn tình huống, và app **đọc to từng bước bình tĩnh** cho bạn làm theo — kèm nhịp metronome để ép tim đúng nhịp.

### Điểm cốt lõi cần nhớ
- Đây là một **PWA (Progressive Web App)**: cài được lên màn hình chính như app thật và **chạy offline 100%**.
- **Không backend, không framework, không build, không thư viện ngoài** — 100% HTML/CSS/JS thuần.
- Hai "nửa": **nửa giải trí** (mini-game học) và **nửa công cụ thật** (Panic Mode, máy đếm nhịp CPR, số khẩn cấp, thẻ y tế).
- **Song ngữ Việt/Anh**, dữ liệu lưu cục bộ (localStorage), không gửi đi đâu.

## 1.2. Dự án giải quyết vấn đề gì?

1. **Kiến thức sơ cứu khô khan, khó nhớ** → biến thành trò chơi vui để não ghi nhớ tự nhiên.
2. **Lúc khẩn cấp con người hoảng loạn, quên hết** → Chế độ Hoảng loạn đọc to từng bước, có nút gọi 115 luôn hiện.
3. **Mất mạng lúc cần nhất** → PWA offline đảm bảo dùng được kể cả không có Internet.
4. **Trình diễn năng lực kỹ thuật**: làm chủ hàng loạt Web API trình duyệt mà không cần thư viện, kiến trúc offline-first, i18n, accessibility — đây là phần "ăn điểm" khi phỏng vấn.

> 💡 **Mẹo phỏng vấn**: Khi được hỏi "dự án này để làm gì", hãy nhấn vào **sự giao thoa giữa giá trị xã hội và độ khó kỹ thuật**: "Em muốn làm một thứ vừa hữu ích thật (cứu người) vừa thử thách kỹ thuật — đặc biệt là yêu cầu **offline-first** cho một app cấp cứu, và làm mọi thứ bằng vanilla JS không phụ thuộc."

## 1.3. Các tính năng chính

| Nhóm | Tính năng |
|------|-----------|
| **Học qua chơi** | 17 mini-game (5 kiểu: nhịp/đẩy/vòi nước/ép/trắc nghiệm), tối đa 3 sao mỗi màn, bài học đời thực sau khi chơi |
| **Chế độ Hoảng loạn** | 17 tình huống, đọc to từng bước (TTS), metronome cho CPR, nút gọi 115 |
| **Công cụ khẩn cấp** | Máy đếm nhịp CPR (110/phút), gọi khẩn cấp 1 chạm (115/113/114/111), thẻ y tế cá nhân |
| **Cẩm nang tra cứu** | 17 tình huống dạng accordion, đọc lướt nhanh |
| **Chế độ Survival** | Trắc nghiệm liên tục có đồng hồ đếm ngược, lưu kỷ lục |
| **Thành tích & giữ chân** | Sao, 9 huy hiệu, chuỗi ngày (streak), nhắc ôn tập |
| **Âm thanh & rung** | Tự tổng hợp bằng Web Audio (không file media), rung theo nhịp |
| **Chia sẻ** | Vẽ thẻ ảnh canvas + Web Share API, Open Graph cho link preview |
| **Đa ngôn ngữ** | Việt/Anh toàn bộ giao diện + nội dung + giọng đọc |
| **Khả năng tiếp cận** | Tương phản cao, chữ lớn, giảm chuyển động, chế độ nghiêm túc, ARIA |
| **PWA & offline** | Cài lên màn hình chính, 2 shortcut, chạy offline 100% |

## 1.4. Công nghệ sử dụng (tech stack)

**Cốt lõi:**
- **HTML5 + CSS3 + JavaScript thuần (vanilla, ES6+)** — không React/Vue, không jQuery, không thư viện nào.
- **Không build step**: mở `index.html` là chạy (khuyến nghị chạy qua HTTP để PWA/giọng nói/cảm biến hoạt động đúng).

**Các Web API trình duyệt được dùng:**
- **Service Worker + Cache API** — caching để chạy offline.
- **Web App Manifest** — khai báo PWA (cài lên màn hình chính, icon, shortcut).
- **Web Audio API** — tự tổng hợp toàn bộ âm thanh (không file media).
- **Web Speech API (SpeechSynthesis)** — đọc to từng bước trong Panic Mode (TTS).
- **Vibration API** — rung theo nhịp ép tim.
- **DeviceOrientation API** — cảm biến nghiêng cho game Heimlich.
- **Notifications API** — nhắc ôn tập.
- **Web Share API + Canvas** — chia sẻ thẻ ảnh kết quả.
- **localStorage** — lưu sao, thống kê, cài đặt, thẻ y tế.

**Công cụ phụ trợ (dev, không phải runtime):**
- **Python + Pillow** — các script `gen_icons.py`, `gen_screens.py`, `gen_og.py`, `gen_gif.py` để sinh ảnh icon/screenshot/Open Graph (chạy 1 lần, không nằm trong app).

**Triển khai:** GitHub Pages (web tĩnh, deploy = đẩy file).

> 💡 **Câu hỏi cài bẫy: "Sao không dùng framework?"** Trả lời tốt: "Yêu cầu lớn nhất là **offline-first và nhẹ tuyệt đối** cho một app cấp cứu — mỗi KB và mỗi phụ thuộc đều là rủi ro. Vanilla JS cho em toàn quyền kiểm soát, **không có bề mặt tấn công chuỗi cung ứng** từ thư viện ngoài, không build phức tạp, và app shell rất nhỏ để cache offline. Đổi lại em phải tự viết router, i18n, state — đó cũng là phần em học được nhiều nhất."

## 1.5. Cách chạy (demo flow)

```bash
# Không cần cài gì. Chạy qua HTTP để PWA/giọng nói/cảm biến hoạt động:
cd cap-cuu-101
python -m http.server 8000
# Mở http://localhost:8000
```
> Service worker **chỉ chạy qua `http(s)://`** (localhost hoặc GitHub Pages), không chạy khi mở bằng `file://`.

Kiểm tra cú pháp (vì không có test tự động):
```bash
node --check app.js
node --check sound.js
node --check sw.js
python -m json.tool manifest.json   # validate JSON
```

## 1.6. Cấu trúc thư mục

```
cap-cuu-101/
├─ index.html   # 10 màn (section.screen) + overlay splash/tutorial/settings
├─ style.css    # giao diện tối, cartoon (~590 dòng)
├─ sound.js     # window.Sfx — Web Audio synth + cài đặt âm thanh (~150 dòng)
├─ app.js       # toàn bộ logic: i18n, router, dữ liệu, engine game, Panic, công cụ (~2.980 dòng)
├─ sw.js        # service worker: precache app shell → offline (~80 dòng)
├─ manifest.json# khai báo PWA
├─ icon.svg, icons/, screenshots/   # tài nguyên ảnh
├─ gen_*.py     # script Python (Pillow) sinh ảnh — chỉ dùng lúc dev
└─ *.md         # README, FEATURES, DOCUMENTATION, CONTRIBUTING, MEDICAL_REVIEW...
```

Quy mô: ~4.200 dòng tổng (app.js ~2.980, style.css ~590, sound.js ~150, index.html ~390, sw.js ~80).

**Thứ tự nạp script:** `index.html` nạp `sound.js` **trước** (lộ ra `window.Sfx`), rồi `app.js` (bọc toàn bộ trong một IIFE).

> ⚠️ **Disclaimer quan trọng (nên chủ động nêu):** Đây là sản phẩm giải trí + ghi nhớ, **không thay thế** khoá sơ cứu chính quy hay nhân viên y tế. Việc thật: **gọi 115 trước tiên**. Dự án có hẳn file `MEDICAL_REVIEW.md` làm checklist cho nhân viên y tế rà soát nội dung — thể hiện ý thức trách nhiệm với một app y tế.
