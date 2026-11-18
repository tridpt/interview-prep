# 1. Tổng quan dự án AI Explorer

## 1.1. AI Explorer là gì?

AI Explorer (khẩu hiệu: *Hiểu AI trong 15 phút*) là một **"bảo tàng AI tương tác"** chạy hoàn toàn trên trình duyệt. Người dùng đi qua **26 "phòng"**, mỗi phòng giải thích trực quan một khái niệm cốt lõi về AI và cho **tự tay nghịch** — chứ không chỉ đọc lý thuyết.

Hãy hình dung: bạn bật webcam dạy AI phân biệt vài tư thế trong 30 giây (phòng 01), kéo thanh trượt để xem một mạng nơ-ron uốn ranh giới phân loại (phòng 02), gõ một câu để xem AI cắt nó thành token (phòng 07), rồi tới các phòng "nâng cao" cho lập trình viên như RAG, fine-tuning, agents, và mới nhất là **phòng "AI biết suy nghĩ từng bước"** (chain-of-thought / reasoning models như o1, DeepSeek-R1).

### Điểm cốt lõi cần nhớ
- Là một **PWA (Progressive Web App)**: cài được lên màn hình chính như app thật và **chạy offline**.
- **Không backend, không API key, không framework, không build step** — 100% HTML/CSS/JS thuần với **ES Modules**.
- **Kiến trúc module hoá**: mỗi phòng là một module `.js` độc lập export một hàm `render(root)`; `app.js` chỉ điều phối.
- **Song ngữ Việt/Anh** toàn diện (giao diện + nội dung từng phòng), đổi ngôn ngữ tức thì.
- Giao diện **Neo-brutalism** (viền đen dày, bóng cứng, khối màu rực; font Archivo + Space Mono).
- Dữ liệu (tiến trình, điểm, lộ trình) lưu cục bộ bằng **localStorage**, không gửi đi đâu.

## 1.2. Dự án giải quyết vấn đề gì?

1. **AI bị coi là "hộp đen" khó hiểu, đầy thuật ngữ** → biến từng khái niệm thành một demo tương tác để "thấy tận mắt, nghịch tận tay", không cần công thức toán.
2. **Người mới dễ bị choáng với hơn 20 chủ đề** → có **lộ trình học theo cấp độ** (Người mới / Đầy đủ / Cho lập trình viên) để đi đúng phần mình cần.
3. **Tài liệu học AI thường tĩnh, khô** → mỗi phòng có tương tác, quiz nhỏ kiểm tra hiểu bài, âm thanh phản hồi, và mạch kể chuyện xuyên suốt.
4. **Trình diễn năng lực kỹ thuật**: làm chủ ES Modules, PWA offline-first, i18n tự viết, accessibility (bao gồm tự tính tương phản màu theo chuẩn WCAG), deep-link, và một pipeline CI/CD có test tự chặn deploy — tất cả bằng vanilla JS không phụ thuộc.

> 💡 **Mẹo phỏng vấn**: Khi được hỏi "dự án này để làm gì", hãy nhấn vào **sự giao thoa giữa giá trị giáo dục và độ khó kỹ thuật**: "Em muốn làm một thứ giúp người mới *hiểu AI bằng trực giác* — mỗi khái niệm là một trò chơi nhỏ — và đồng thời tự thử thách mình xây dựng mọi thứ bằng vanilla JS: router, i18n, PWA, deep-link, và cả một hệ thống test tự động."

## 1.3. Các tính năng chính

| Nhóm | Tính năng |
|------|-----------|
| **26 phòng tương tác** | Mỗi phòng một khái niệm AI + một tông màu chủ đề; tự tay nghịch (webcam, kéo trượt, gõ chữ, chọn đáp án…) |
| **Lộ trình học** | *Người mới* (5 phòng cốt lõi), *Đầy đủ*, *Cho lập trình viên* — không bị choáng |
| **Quiz nhỏ mỗi phòng** | "Kiểm tra hiểu bài" ở cuối phòng, cộng dồn vào điểm hành trình |
| **Quiz tổng kết** | Phòng cuối + huy hiệu theo cấp, tải được thành ảnh PNG |
| **Tìm nhanh phòng** | Bấm `/` hoặc 🔍 để tra theo tên/khái niệm, có **bẫy focus** cho hộp thoại (a11y) |
| **Deep-link theo trạng thái** | Một số phòng mã hoá trạng thái vào URL (câu ở Tokenizer/Attention, bài ở Reasoning…) để chia sẻ đúng một demo |
| **Nút Chia sẻ** | Web Share API trên điện thoại, tự copy link (kèm trạng thái) trên desktop |
| **Song ngữ VN/EN** | Toàn bộ giao diện + nội dung, đổi tức thì (không tải lại trang) |
| **PWA & offline** | Cài như app, chạy offline, có thông báo "Đã có bản mới — Tải lại" |
| **Âm thanh & confetti** | Web Audio tự tổng hợp (không file media), confetti khi hoàn thành |
| **Accessibility** | Tôn trọng `prefers-reduced-motion`, ARIA, tự tính tương phản màu WCAG cho chữ |
| **Trình chiếu** | Chế độ toàn màn hình, điều hướng bằng phím ←/→ |

## 1.4. Công nghệ sử dụng (tech stack)

**Cốt lõi:**
- **HTML5 + CSS3 + JavaScript thuần (vanilla, ES6+)** với **ES Modules** (`import`/`export`) — không React/Vue, không jQuery, không thư viện runtime.
- **Không build step**: mở qua HTTP server là chạy (ES modules bắt buộc phải phục vụ qua `http(s)://`, không mở `file://`).

**Các Web API / kỹ thuật trình duyệt được dùng:**
- **Service Worker + Cache API** — caching để chạy offline (PWA).
- **Web App Manifest** (`manifest.json`) — khai báo PWA (cài lên màn hình chính, icon, màu chủ đề).
- **URL hash + `hashchange`** — router phía client, không tải lại trang.
- **`history.replaceState`** — deep-link trạng thái phòng vào query của hash.
- **localStorage** — lưu tiến trình, điểm quiz, câu đã đúng, lộ trình, ngôn ngữ.
- **Web Audio API** — tự tổng hợp âm thanh phản hồi (không file media).
- **Canvas API** — vẽ mạng nơ-ron, diffusion, confetti, thẻ ảnh chia sẻ.
- **getUserMedia (webcam)** — phòng "Tự tay dạy AI".
- **`matchMedia('(prefers-reduced-motion)')`** — accessibility.
- **Web Share API + Clipboard API** — chia sẻ phòng.

**Kiểm thử & CI:**
- **Playwright** (Chromium) cho smoke test.
- Hai script kiểm tra thuần Node (không cần browser): `check-assets.mjs` (toàn vẹn cache), `i18n.mjs` (toàn vẹn song ngữ).
- **GitHub Actions** chạy toàn bộ test trước mỗi lần deploy; test fail thì **chặn deploy**.

**Công cụ phụ trợ (dev, không phải runtime):**
- **Python + Pillow** — `gen_icons.py`, `gen_og.py` sinh ảnh icon/Open Graph (chạy một lần).

**Triển khai:** GitHub Pages (web tĩnh, deploy = đẩy file qua workflow).

> 💡 **Câu hỏi cài bẫy: "Sao không dùng framework?"** Trả lời tốt: "Mục tiêu là một app **giáo dục nhẹ, offline-first, sống rất lâu** mà không phụ thuộc thư viện nào — mỗi khái niệm AI chỉ cần một canvas và ít JS thuần. Vanilla + ES Modules cho em app shell nhỏ để cache offline, **không có bề mặt tấn công chuỗi cung ứng**, không build phức tạp, và mỗi phòng là một module cô lập dễ thêm/bớt. Đổi lại em phải tự viết router, i18n, PWA update flow — đó cũng là phần học được nhiều nhất."

## 1.5. Cách chạy (demo flow)

Dự án dùng ES modules nên **bắt buộc chạy qua một HTTP server cục bộ** (không mở trực tiếp `file://`).

```bash
# Cách 1 — Python (có sẵn trên hầu hết máy):
cd d:\AI_App\ai-explorer
python -m http.server 8000
# Mở http://localhost:8000

# Cách 2 — VS Code: cài "Live Server", chuột phải index.html → Open with Live Server.
```
> Lưu ý: phòng 01 (webcam) cần quyền camera và phải chạy qua `localhost` hoặc HTTPS — yêu cầu bảo mật của trình duyệt.

Chạy kiểm thử:
```bash
npm install
npx playwright install chromium
npm run check-assets   # kiểm tra toàn vẹn cache offline (nhanh, không cần browser)
npm run test:i18n      # kiểm tra mọi chuỗi { vi, en } đủ hai vế
npm run serve          # cửa sổ 1: server tại http://localhost:8000
npm run test:all       # cửa sổ 2: assets, i18n, SEO, smoke, focus, PWA, a11y
```

## 1.6. Cấu trúc thư mục

```
ai-explorer/
├─ index.html          # Khung trang + meta chia sẻ + favicon
├─ style.css           # Toàn bộ giao diện (Neo-brutalism)
├─ app.js              # Router + điều hướng + mảng ROOMS + toolbar + tìm phòng + chia sẻ + i18n glue
├─ i18n.js             # Song ngữ VN/EN (tx, getLang, setLang)
├─ store.js            # localStorage: tiến trình, điểm quiz, lộ trình
├─ roomstate.js        # Deep-link: đọc/ghi trạng thái phòng vào query của hash
├─ roomquiz.js         # Ngân hàng quiz nhỏ + chèn "kiểm tra hiểu bài" mỗi phòng
├─ tracks.js           # Định nghĩa lộ trình (Người mới / Đầy đủ / Cho lập trình viên)
├─ sound.js            # Âm thanh Web Audio + confetti
├─ analytics.js        # Thống kê ẩn danh (GoatCounter, tuỳ chọn)
├─ pwa.js              # Đăng ký service worker + nút cài app + thông báo bản mới
├─ sw.js               # Service worker: cache offline (ASSETS + số phiên bản CACHE)
├─ manifest.json       # Khai báo PWA
├─ rooms/              # Mỗi phòng là một module độc lập export render(root)
│   ├─ home.js         # Trang chủ + chọn lộ trình + tiến trình
│   ├─ teachable.js    # Webcam KNN
│   ├─ neural-net.js   # MLP 2→H→1 huấn luyện trên trình duyệt
│   ├─ … (23 phòng khái niệm) …
│   ├─ reasoning.js    # Chain-of-thought: trả lời ngay vs suy nghĩ từng bước (PHÒNG MỚI)
│   └─ summary.js      # Tổng kết + quiz + huy hiệu
├─ data/embeddings.js  # Vector 2D tính sẵn cho các từ
└─ tests/              # check-assets, i18n, SEO, smoke, focus, PWA, a11y
```

> 💡 **Điểm ăn điểm để nêu chủ động:** "`rooms-meta.js` giữ metadata cho cả app và trình prerender SEO; `room-loaders.js` lazy-load code tương tác khi vào phòng. Thêm phòng cần khai báo metadata, loader, module và asset offline. `npm run test:all` kiểm tra cả SEO, i18n, smoke, a11y và PWA."
