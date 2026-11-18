# 4. Code chi tiết — đi qua từng file

File này giải thích **vai trò và các hàm chính** của từng file mã nguồn. Đọc để tự tin khi bị "soi code".

---

## 4.1. `app.js` — bộ điều phối trung tâm

File lớn nhất, gồm:

- **Import `ROOM_META` và `ROOM_LOADERS`**, chỉ nạp mã từng phòng khi truy cập; import tĩnh các hệ con (i18n, store, sound, analytics, roomstate, roomquiz).
- **`export const ROOMS`** — mảng 26 phòng, mỗi phòng có `id/icon/num/title/question/blurb/render` (song ngữ).
- **`UI`** — chuỗi giao diện chung (nút, tiêu đề, ô tìm) dạng `{vi,en}`.
- **`SEARCH_KEYWORDS`** — từ khoá tìm kiếm song ngữ cho mỗi phòng.
- **`THEME`** — bộ màu `[accent, accent2, rgb]` cho mỗi phòng.
- **WCAG helpers**: `luminance(hex)`, `contrast(l1,l2)`, `darkenForText(hex)` — tự làm đậm màu chữ tới khi đạt tương phản 4.5:1.
- **`applyTheme(id)`** — đặt biến CSS màu theo phòng.
- **`HINTS` + `showHint(id)`** — gợi ý onboarding hiện 1 lần/phòng, tự ẩn sau 7s.
- **`openSearch()`** — bảng tìm nhanh phòng (phím `/`): lọc trên `ROOMS` + `SEARCH_KEYWORDS`, hỗ trợ ↑↓/Enter/Esc, và **bẫy focus** (Tab quẩn trong dialog, trả focus về nút mở khi đóng). Chuẩn hoá tiếng Việt bằng `normalize("NFD")` để gõ không dấu vẫn ra.
- **`shareRoom(room)`** — Web Share API + fallback clipboard, mang theo trạng thái deep-link.
- **`toast(msg)`** — thông báo nhỏ tự biến mất.
- **`buildToolbar()`** — dựng nút tìm/ngôn ngữ/âm thanh/trình chiếu.
- **`renderProgress(activeId)`** — dãy chấm tiến trình, có ARIA cho từng phòng.
- **`navigate(id)`** — đổi `location.hash`.
- **`route()`** — hàm điều phối chính (xem file 03, mục 3.4).
- **Các listener**: `hashchange` → `route`; `langchange` → dựng lại toolbar + render lại phòng; phím ←/→ điều hướng; phím `/` mở tìm kiếm; click phát tiếng nhẹ.

---

## 4.2. `i18n.js` — song ngữ

- `getLang()` / `setLang(l)` — đọc/đổi ngôn ngữ (lưu localStorage, phát `"langchange"`).
- `tx(a, b)` — hàm dịch: nhận `{vi,en}` **hoặc** hai chuỗi.
- Bọc localStorage trong `try/catch` (an toàn với Safari private mode).

```js
export function tx(a, b) {
  if (a && typeof a === "object") return a[lang] ?? a.vi;
  return lang === "en" ? b : a;
}
```

---

## 4.3. `store.js` — lưu tiến trình

Một khoá localStorage `"ai-explorer-progress"` chứa `{ visited, bestScore, quizAnswered, track }`. Các hàm:
- `markVisited(id)` / `getVisited()` — phòng đã ghé (trả về `Set`).
- `getBestScore()` / `setBestScore(score)` — điểm quiz cao nhất (trả `true` nếu lập kỷ lục).
- `markMicroQuiz(roomId, qIndex)` / `getMicroSolved(roomId)` / `getMicroTotal()` — quiz nhỏ.
- `getTrack()` / `setTrack(track)` — lộ trình đang chọn.
- `resetProgress()` — xoá sạch.

Mọi ghi bọc `try/catch`, mọi đọc có giá trị mặc định an toàn.

---

## 4.4. `roomstate.js` — deep-link

- `parseHash(hash)` → `{ id, params }` (tách phòng và query).
- `getParams()` / `getParam(key, fallback)` — đọc tham số.
- `setParams(obj)` — ghi tham số bằng `history.replaceState` (bỏ qua giá trị rỗng/null; **không** phát `hashchange`).
- `buildShareUrl(id, obj)` — URL tuyệt đối để chia sẻ.

---

## 4.5. `roomquiz.js` — quiz "kiểm tra hiểu bài"

- `BANK` — ngân hàng câu hỏi, mỗi phòng 1 câu: `{ q:{vi,en}, opts:{vi:[],en:[]}, correct }`.
- `hasMicroQuiz(roomId)` — phòng có quiz không.
- `renderMicroQuiz(container, roomId)` — chèn khối quiz vào cuối phòng: hiển thị đáp án, chấm đúng/sai, phát âm thanh, tô sáng đáp án đúng, và ghi điểm qua `markMicroQuiz`. Câu đã đúng trước đó thì khoá lại và đánh dấu sẵn.

---

## 4.6. `tracks.js` — lộ trình học

- `TRACKS` — 3 lộ trình `beginner` / `full` / `dev`, mỗi cái có `icon/label/desc/rooms`.
- `trackRoomIds(trackKey, allRooms)` — trả danh sách id (nếu `rooms=null` thì tất cả).
- `trackLabel(trackKey)` — nhãn đã dịch.

---

## 4.7. `sound.js` — âm thanh & confetti

- Tự tổng hợp âm bằng **Web Audio**: hàm `tone(freq, dur, type, gain, when)` ghép `OscillatorNode` + `GainNode` — không cần file media.
- `sfx` — bộ hiệu ứng: `click / pop / tick / success / wrong / complete`.
- `isMuted()` / `setMuted(v)` — bật/tắt (lưu localStorage).
- `celebrate()` — vẽ confetti bằng Canvas khi hoàn thành.
- `ac()` xử lý **autoplay policy**: chỉ tạo/`resume()` `AudioContext` khi cần (sau tương tác người dùng).

---

## 4.8. `pwa.js` — service worker & cập nhật

- Đăng ký `sw.js` khi trang load.
- Theo dõi worker mới (`updatefound` → `statechange`); khi đã `installed` mà có controller cũ → gọi `notifyUpdate()` hiện toast **"Đã có bản mới — Tải lại"** (song ngữ theo `<html lang>`).
- Bấm "Tải lại" → gửi `postMessage("SKIP_WAITING")`; khi worker mới chiếm quyền (`controllerchange`) → tự reload một lần.
- Xử lý `beforeinstallprompt` → hiện nút ⬇ cài app; `appinstalled` → gỡ nút.

---

## 4.9. `sw.js` — service worker

- `const CACHE = "ai-explorer-v22";` — số phiên bản cache.
- `ASSETS` — danh sách mọi file cần cache offline (mọi module `.js`, `index.html`, `style.css`, icon…).
- `install` → precache `ASSETS` (KHÔNG tự `skipWaiting`).
- `message` → nếu nhận `"SKIP_WAITING"` thì `skipWaiting()`.
- `activate` → xoá cache cũ khác `CACHE`, rồi `clients.claim()`.
- `fetch` → **cache-first** cho origin nội bộ, đi mạng cho phần còn lại (vd Google Fonts), fallback `index.html` khi offline.

---

## 4.10. Cấu trúc một file phòng điển hình

Mọi phòng theo cùng khuôn mẫu, ví dụ rút gọn từ `neural-net.js`:

```js
import { sfx } from "../sound.js";
import { tx } from "../i18n.js";

export function roomNeuralNet(root) {
  // 1) Dựng HTML nội dung (song ngữ qua tx), thường có .panel, .row, .takeaway
  root.innerHTML = `… <canvas id="nnCanvas"></canvas> …
    <div class="takeaway">${tx("💡 Điều cốt lõi: …", "💡 Key idea: …")}</div>`;

  // 2) Lấy phần tử, khởi tạo trạng thái cục bộ
  const canvas = root.querySelector("#nnCanvas");
  let timer = null;

  // 3) Gắn sự kiện tương tác (nút, thanh trượt…)
  root.querySelector("#trainBtn").onclick = () => { /* … */ };

  // 4) Dọn dẹp khi rời phòng (QUAN TRỌNG để tránh rò rỉ)
  window.addEventListener("roomleave", () => { if (timer) clearInterval(timer); }, { once: true });
}
```

Đặc điểm chung đáng nêu:
- **Mọi chuỗi hiển thị đi qua `tx`** → song ngữ.
- **Tự dọn tài nguyên** qua `roomleave` (timer, webcam, animation) — vì không có framework quản lý vòng đời.
- **Tính toán tại chỗ** (KNN, MLP, k-means, bigram…), không gọi mạng → chạy offline.

**Ví dụ webcam (`teachable.js`)** — KNN thuần: rút đặc trưng ảnh về lưới 24×24 xám (`extractFeature`), phân loại theo hàng xóm gần nhất (`distance`), và khi rời phòng thì tắt camera:
```js
window.addEventListener("roomleave", () => {
  if (predTimer) clearInterval(predTimer);
  if (stream) stream.getTracks().forEach((t) => t.stop());
}, { once: true });
```

**Ví dụ mạng nơ-ron (`neural-net.js`)** — MLP 2→H→1 tự cài `forward`/`trainStep` (lan truyền ngược thủ công), vẽ ranh giới phân loại lên canvas theo thời gian thực khi kéo số nơ-ron.

Xem `reasoning.js` được mổ xẻ chi tiết trong file `08-code-walkthrough.md`.
