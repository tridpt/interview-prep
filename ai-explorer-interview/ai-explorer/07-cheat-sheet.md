# 7. Cheat Sheet — Xem nhanh trước giờ G

Một trang tóm tắt. Đọc lướt 5-10 phút trước khi vào phỏng vấn.

---

## 🎯 Pitch 30 giây (học thuộc)
> AI Explorer: "bảo tàng AI tương tác" gồm **26 phòng** chạy 100% trên trình duyệt, mỗi phòng minh hoạ một khái niệm AI (mạng nơ-ron, token, embeddings, attention, RAG, agents, RLHF, **chain-of-thought**…) và cho tự tay nghịch. **Vanilla JS + ES Modules — không backend, không API key**, là **PWA chạy offline**, **song ngữ VN/EN**, giao diện Neo-brutalism. Kiến trúc **module hoá**: mỗi phòng một file export `render(root)`, thêm vào mảng `ROOMS` là xong; CI 3 lớp test **chặn deploy nếu fail**.

## 🧱 Tech stack
- **Cốt lõi**: Vanilla HTML/CSS/JS (ES6+) + **ES Modules**, không build step.
- **PWA**: Web App Manifest + Service Worker + Cache API.
- **Web APIs**: hash routing + `hashchange`, `history.replaceState` (deep-link), localStorage, Web Audio (synth), Canvas, getUserMedia, Web Share/Clipboard, `matchMedia(prefers-reduced-motion)`.
- **Test/CI**: Playwright (smoke), Node scripts (check-assets, i18n), GitHub Actions → GitHub Pages.
- **Dev tools**: Python + Pillow (gen_icons.py, gen_og.py — không phải runtime).

## 🗂️ Các file
| File | Vai trò |
|---|---|
| `app.js` | Điều phối: `ROOMS`, router `hashchange`, `route()`, THEME, toolbar, tìm phòng, chia sẻ |
| `i18n.js` | Song ngữ: `tx({vi,en})` / `tx(a,b)`, `setLang` phát `langchange` |
| `store.js` | localStorage `ai-explorer-progress`: visited, bestScore, quizAnswered, track |
| `roomstate.js` | Deep-link: `parseHash`, `getParam`, `setParams` (replaceState), `buildShareUrl` |
| `roomquiz.js` | `BANK` + `renderMicroQuiz`, `hasMicroQuiz` |
| `tracks.js` | `TRACKS` (beginner/full/dev) + `trackRoomIds` |
| `sound.js` | Web Audio synth (`sfx`) + `celebrate()` confetti |
| `pwa.js` | Đăng ký SW + toast "bản mới" + nút cài app |
| `sw.js` | `CACHE="ai-explorer-v22"` + `ASSETS`, cache-first, không tự skipWaiting |
| `rooms/*.js` | Mỗi phòng export `render(root)`; dọn qua `roomleave` |

## 📐 Kiến trúc
SPA phía client, **không backend**. `app.js` khung — phòng lo ruột. `ROOMS` = nguồn sự thật duy nhất (mạch kể chuyện). Router bằng hash. Mỗi phòng tính AI tại chỗ (KNN/MLP/k-means/bigram/mô phỏng) → offline.

## 🌊 Service Worker (nhớ kỹ)
- **install** → precache `ASSETS` (KHÔNG tự skipWaiting).
- **message** → nhận `SKIP_WAITING` thì `skipWaiting()`.
- **activate** → xoá cache cũ ≠ `CACHE` → `clients.claim()`.
- **fetch** → cache-first origin nội bộ; đi mạng phần còn lại; fallback `index.html`.
- Đổi asset → **bump `CACHE`** (v22→v23). SW chỉ chạy qua `http(s)://`.
- Cập nhật: toast "Đã có bản mới — Tải lại" → `SKIP_WAITING` → `controllerchange` → reload.

## 🧩 ES Modules
`import`/`export`; mỗi phòng một module. Phải chạy qua HTTP (fetch chặn `file://`). Không cần bundler.

## 🌐 i18n
`lang` (localStorage) + `tx()`; nội dung song ngữ ngay trong dữ liệu ({vi,en}); đổi ngôn ngữ → `langchange` → dựng lại toolbar + render lại phòng. Test `i18n.mjs` quét mọi {vi,en}.

## 🔗 Deep-link
Trạng thái vào query của hash (`#/reasoning?p=2`); `setParams` dùng `replaceState` (không render lại); nút Chia sẻ mang theo URL (Web Share/clipboard).

## ♿ Accessibility
`prefers-reduced-motion`; ARIA (dialog/listbox/current); **bẫy focus** ô tìm + trả focus khi đóng; điều hướng phím; **tự tính tương phản WCAG**: `luminance`+`contrast`+`darkenForText` → `--accent-ink` ≥ 4.5:1.

## 🧪 Testing & CI (chặn deploy nếu fail)
1. `check-assets.mjs` — mọi asset runtime phải nằm trong `ASSETS` (chống offline hỏng thầm).
2. `i18n.mjs` — mọi `{vi,en}` đủ hai vế (bộ duyệt ký tự, không regex).
3. `smoke.mjs` — Playwright duyệt mọi phòng (đọc `ROOMS`), fail nếu lỗi console.
4. `focus-trap.mjs`, `pwa.mjs`.

## 💭 Phòng Reasoning (điểm nhấn — nhớ kỹ)
- Phòng 25, chain-of-thought / reasoning models (o1, DeepSeek-R1).
- **Trả lời ngay** (sai, đỏ, ≈1–3 token) vs **Suy nghĩ từng bước** (đúng, xanh, ≈48–70 token).
- Ý cốt lõi: model không "thông minh hơn", chỉ có **thêm không gian để tính** — mỗi bước là ngữ cảnh cho bước sau. Đánh đổi: chậm hơn + tốn token.
- CoT = kỹ thuật prompt; reasoning model = mô hình huấn luyện để tự nghĩ.
- Deterministic, song ngữ, deep-link `p`, có quiz, thuộc lộ trình *Cho lập trình viên*.

## 🤖 Khái niệm AI theo phòng (một dòng)
KNN (dạy từ ví dụ) · MLP (deep learning) · overfitting (học vẹt) · decision tree (minh bạch) · RL (thưởng/phạt) · k-means (tự gom nhóm) · token · embeddings (bản đồ nghĩa) · attention · next-token/hallucination · diffusion · gợi ý · thiên kiến · adversarial · turing · chatbot · RAG (tra tài liệu) · prompting vs fine-tuning · agents (tool-use) · multimodal · context window · prompt injection · RLHF · năng lượng · **chain-of-thought**.

## ⚖️ Trade-off chính
No-framework/no-build → nhẹ, offline tốt, no-deps; đổi lại tự viết router/i18n/PWA + tự quản vòng đời phòng (bù bằng `roomleave` + test).

## ⚠️ Giới hạn đã biết (chủ động nêu)
Các "AI" là mô phỏng/thuật toán nhỏ để dạy trực giác (không phải model thật); webcam cần quyền + localhost/https; chưa có unit test cho từng thuật toán phòng.

## 💬 Thuật ngữ chốt
- **PWA / Service Worker / Cache API / cache versioning / app shell**
- **ES Modules / hash routing / history.replaceState / deep-link**
- **i18n / langchange / localStorage**
- **Web Audio / autoplay policy / Canvas / getUserMedia**
- **prefers-reduced-motion / ARIA / focus trap / WCAG contrast**
- **chain-of-thought / reasoning models / token trade-off**
- **RAG / embeddings / attention / RLHF / fine-tuning vs prompting / agents**

---

### ✅ Checklist tâm lý trước phỏng vấn
- [ ] Pitch dự án trong 30 giây.
- [ ] Giải thích vòng đời service worker + luồng cập nhật không kẹt bản cũ.
- [ ] Lý do chọn vanilla JS + ES Modules (offline-first, no-deps).
- [ ] Giải thích chain-of-thought & đánh đổi token.
- [ ] Nêu 3 lớp test và vì sao `check-assets` quan trọng.
- [ ] Kể cách thêm một phòng mới (mở rộng).
- [ ] Chuẩn bị 2 câu hỏi ngược lại nhà tuyển dụng.

**Hít thở sâu. Bạn hiểu dự án này. Cứ tự tin kể câu chuyện của mình. 🍀**
