# 2. Các khái niệm nền tảng (lý thuyết)

File này giải thích **mọi thuật ngữ** trong dự án, viết cho người chưa biết gì. Chia làm hai phần: **(A) khái niệm web/kỹ thuật** mà dự án dùng để xây, và **(B) khái niệm AI** mà các phòng minh hoạ. Nắm chắc phần này sẽ giúp bạn tự tin trước mọi câu hỏi "khái niệm".

---

# PHẦN A — Khái niệm web & kỹ thuật

## 2.1. ES Modules (import / export)

**Là gì?** Hệ thống module chuẩn của JavaScript hiện đại: mỗi file `.js` là một module, chia sẻ code bằng `export` và `import`. Trong dự án, `app.js` khai báo `<script type="module">`, rồi:

```js
import { renderHome } from "./rooms/home.js";
import { getLang, setLang, tx } from "./i18n.js";
export const ROOMS = [ /* … */ ];
```

**Vì sao cần HTTP server (không mở `file://`)?** Trình duyệt tải module qua `fetch`, mà `fetch` bị chặn với giao thức `file://` vì lý do bảo mật (CORS). Vì vậy phải chạy qua `http://localhost` (vd `python -m http.server`).

**Lợi ích với dự án:** mỗi phòng là một module cô lập, không rò biến ra global, phụ thuộc rõ ràng qua `import`. Không cần bundler/build — trình duyệt tự nạp.

> 💡 So với dự án "một file IIFE khổng lồ", đây là cách hiện đại hơn: ES Modules cho cô lập scope *và* khả năng tách file, mà vẫn không cần build step.

---

## 2.2. PWA (Progressive Web App)

**PWA là gì?** Một web app được "nâng cấp" để hành xử gần như app native: cài được lên màn hình chính, có icon riêng, chạy toàn màn hình, và **chạy offline**.

**Ba thành phần làm nên PWA (đủ cả trong dự án):**
1. **HTTPS** (hoặc localhost) — bắt buộc để đăng ký service worker.
2. **Web App Manifest** (`manifest.json`) — JSON khai báo tên, icon, màu chủ đề, chế độ hiển thị.
3. **Service Worker** (`sw.js`) — script chạy nền, chặn request để phục vụ từ cache → offline.

**Trong dự án:** PWA giúp AI Explorer cài được như app học AI, và **chạy trọn vẹn khi offline** — mọi phòng đều là JS/canvas tính toán tại chỗ, không gọi mạng.

---

## 2.3. Service Worker & Cache API & cache versioning

**Service Worker (SW) là gì?** Một file JavaScript chạy ở "tầng nền", tách biệt khỏi trang, đóng vai trò **proxy mạng phía client**: chặn mọi request và quyết định trả từ cache hay đi mạng.

**Vòng đời (lifecycle):**
- **install** — precache (tải sẵn) danh sách `ASSETS` vào Cache API.
- **activate** — dọn cache cũ có version khác.
- **fetch** — mỗi request đi qua đây; dự án dùng **cache-first** cho asset nội bộ (trả cache nếu có, không thì đi mạng rồi cache lại), fallback `index.html` khi offline.

**Cache versioning — kỹ thuật quan trọng nhất:** dự án dùng hằng `const CACHE = "ai-explorer-v22";`. Mỗi lần đổi file phải **bump (tăng)** số version (v22 → v23). Khi SW mới `activate`, nó xoá mọi cache có tên khác `CACHE`, nạp bản mới. Nhờ vậy người dùng không kẹt ở bản cũ.

**Điểm tinh tế trong dự án — không tự `skipWaiting`:**
```js
self.addEventListener("install", (e) => {
  // KHÔNG tự skipWaiting: để worker mới "chờ" tới khi người dùng bấm "Tải lại".
  e.waitUntil(caches.open(CACHE).then((c) => c.addAll(ASSETS)));
});
self.addEventListener("message", (e) => {
  if (e.data === "SKIP_WAITING") self.skipWaiting();
});
```
SW mới cố tình **chờ** thay vì chiếm quyền ngay, để không đổi asset giữa chừng khi người dùng đang dùng. Chỉ khi họ bấm "Tải lại" trên toast (pwa.js gửi message `SKIP_WAITING`), worker mới mới kích hoạt và trang tự reload.

> 💡 **Câu hỏi hay gặp: "Vì sao SW không chạy với `file://`?"** Vì lý do bảo mật, SW chỉ chạy trên `https://` hoặc `localhost`.

---

## 2.4. Router phía client bằng URL hash

**Là gì?** Đổi "trang" (phòng) mà **không tải lại** toàn bộ trang. Dự án dùng phần **hash** của URL (`#/tokenizer`):

```js
window.addEventListener("hashchange", route);
export function navigate(id) {
  location.hash = id === "home" ? "" : `#/${id}`;
}
```
Đổi hash không làm trình duyệt tải lại trang, chỉ phát sự kiện `hashchange` → hàm `route()` đọc phòng hiện tại và render. Hash cũng là cách tự nhiên để **deep-link** (xem 2.5) mà vẫn hoạt động trên hosting tĩnh như GitHub Pages (không cần cấu hình server rewrite).

---

## 2.5. Deep-link trạng thái (roomstate.js)

**Vấn đề:** mở đúng một phòng thì dễ, nhưng chia sẻ đúng *một demo cụ thể* (câu đang gõ ở Tokenizer, bài đang chọn ở Reasoning) thì cần lưu **trạng thái** vào URL.

**Giải pháp:** mã hoá trạng thái vào **query của hash**, ví dụ `#/reasoning?p=2` hay `#/tokenizer?text=Xin%20chào`. `roomstate.js` cung cấp:
- `parseHash()` → tách hash thành `{ id, params }`.
- `getParam(key, fallback)` → đọc một tham số.
- `setParams(obj)` → ghi tham số bằng `history.replaceState` (KHÔNG kích hoạt điều hướng lại, tránh render lặp).
- `buildShareUrl(id, obj)` → tạo URL tuyệt đối để chia sẻ.

> 💡 Điểm hay: dùng `replaceState` (không `hashchange`) để cập nhật URL "âm thầm" khi người dùng gõ/chọn, nên không gây render lại phòng.

---

## 2.6. i18n (đa ngôn ngữ) tự viết

**i18n là gì?** Thiết kế app để hỗ trợ nhiều ngôn ngữ (viết tắt vì giữa "i" và "n" của *internationalization* có 18 chữ).

**Cơ chế trong dự án (`i18n.js`, không thư viện):**
- Biến `lang` đọc từ `localStorage` (mặc định `"vi"`).
- Hàm `tx(obj)` — nhận object `{ vi, en }`, trả chuỗi theo ngôn ngữ hiện tại.
- Hàm `tx(a, b)` — dạng hai tham số, chọn nhanh giữa hai chuỗi.
- `setLang(l)` — đổi ngôn ngữ, lưu localStorage, phát sự kiện `"langchange"`.

```js
export function tx(a, b) {
  if (a && typeof a === "object") return a[lang] ?? a.vi; // tx({vi,en})
  return lang === "en" ? b : a;                            // tx("vi","en")
}
```

Khi `langchange` phát ra, `app.js` dựng lại toolbar và **render lại phòng hiện tại** để mọi chuỗi động đổi theo. Toàn bộ nội dung song ngữ được lưu ngay trong cấu trúc dữ liệu (mỗi phòng: `title/question/blurb` là `{vi,en}`).

> 💡 Có hẳn một test (`i18n.mjs`) quét mọi object `{vi,en}` để bắt lỗi quên điền một vế — xem file 05.

---

## 2.7. localStorage & lưu trạng thái

`store.js` gói toàn bộ việc đọc/ghi localStorage dưới một khoá `"ai-explorer-progress"` (JSON):
- `visited` — danh sách phòng đã ghé.
- `bestScore` — điểm quiz tổng kết cao nhất.
- `quizAnswered` — `{ roomId: [chỉ số câu nhỏ đã đúng] }`.
- `track` — lộ trình đang chọn (`beginner`/`full`/`dev`).

Mọi hàm đọc/ghi bọc trong `try/catch` để không vỡ ở chế độ riêng tư (Safari private mode chặn localStorage).

---

## 2.8. Canvas, accessibility & một số thuật ngữ khác

- **Canvas API**: vẽ đồ hoạ 2D bằng code — dùng cho mạng nơ-ron (vẽ ranh giới phân loại), diffusion, confetti.
- **`prefers-reduced-motion`**: tín hiệu hệ điều hành báo người dùng muốn giảm hiệu ứng; các phòng có animation (vd Reasoning hiện từng bước) kiểm tra và tắt độ trễ khi bật.
- **Bẫy focus (focus trap)**: giữ Tab/Shift+Tab quẩn trong hộp thoại tìm kiếm và trả focus về nút mở khi đóng — chuẩn a11y cho modal.
- **ARIA**: thuộc tính giúp trình đọc màn hình hiểu giao diện (`role="dialog"`, `aria-modal`, `aria-current`, `aria-label`…).
- **Tương phản WCAG tự tính**: `app.js` có hàm `luminance()`, `contrast()` và `darkenForText()` để **làm đậm dần màu chủ đề** tới khi đạt tỉ lệ tương phản ≥ 4.5:1 với nền sáng — đảm bảo chữ màu vẫn đọc được (xem file 05).
- **App shell**: bộ khung tối thiểu (HTML/CSS/JS) được precache để chạy offline.
- **Web Audio synthesis**: tạo âm bằng oscillator + gain, không cần file media.

---

# PHẦN B — Khái niệm AI mà các phòng minh hoạ

Mỗi phòng là một khái niệm. Dưới đây là "vũ khí lý thuyết" để bạn giải thích được nếu người phỏng vấn hỏi "phòng này dạy gì".

| Phòng | Khái niệm AI | Ý một câu |
|-------|--------------|-----------|
| Teachable | **KNN / học từ ví dụ** | AI học từ *dữ liệu mẫu*, không phải luật viết tay; phân loại theo hàng xóm gần nhất |
| Neural-net | **Mạng nơ-ron / MLP** | Nhiều nơ-ron ghép lại tự điều chỉnh trọng số để học hình thù phức tạp (deep learning) |
| Overfitting | **Overfitting** | AI "học vẹt" bài cũ, giỏi train nhưng sai test; cần tổng quát hoá |
| Decision-tree | **Cây quyết định** | Loại AI *minh bạch* — chuỗi câu hỏi có/không, thấy được từng luật |
| Reinforcement | **Học tăng cường (RL)** | Học bằng thưởng/phạt qua thử–sai, không cần đáp án sẵn |
| Clustering | **Học không giám sát / k-means** | Tự gom nhóm dữ liệu giống nhau mà không cần nhãn |
| Tokenizer | **Token & tokenizer** | AI cắt chữ thành *token*, tính chi phí theo token (không theo chữ cái) |
| Embeddings | **Embeddings / vector nghĩa** | Từ → điểm trên bản đồ; khoảng cách mang nghĩa ("vua − đàn ông + đàn bà = nữ hoàng") |
| Attention | **Cơ chế attention** | Mỗi từ "nhìn" vào từ nào khác để hiểu ngữ cảnh (nền tảng Transformer) |
| Next-token | **Đoán từ tiếp theo / hallucination** | AI sinh chữ bằng cách đoán từ kế theo xác suất → đôi khi "ảo giác" |
| Diffusion | **Diffusion (tạo ảnh)** | Từ nhiễu ngẫu nhiên khử nhiễu dần thành ảnh |
| Recommendation | **Hệ gợi ý** | Đoán gu từ hành vi thích/bỏ qua (TikTok/YouTube/Netflix) |
| Bias | **Thiên kiến** | AI học cả định kiến trong dữ liệu con người |
| Adversarial | **Tấn công đối kháng** | Nhiễu vô hình khiến AI nhìn sai — bài học về robustness/an toàn |
| Turing | **Người hay AI viết?** | Rèn con mắt phân biệt văn người/AI |
| Chatbot | **Pipeline LLM** | Ghép token + nghĩa + attention + đoán chữ thành trợ lý |
| RAG | **Retrieval-Augmented Generation** | Tra tài liệu liên quan → đưa vào ngữ cảnh → trả lời có dẫn nguồn, bớt bịa |
| Finetune | **Prompting vs Fine-tuning** | Chỉ dẫn lúc dùng vs huấn luyện lại trọng số — khi nào dùng cái nào |
| Agents | **AI agents / tool-use** | AI lên kế hoạch → gọi công cụ → xem kết quả → lặp cho tới xong |
| Multimodal | **Multimodal** | Ảnh + chữ đưa về cùng "không gian nghĩa" |
| Context-window | **Cửa sổ ngữ cảnh** | AI chỉ nhớ được lượng token giới hạn; token cũ rơi ra → "quên" |
| Prompt-injection | **Prompt injection & an toàn** | "Câu lệnh lén" lừa AI vì lời hệ thống và lời người dùng trộn chung một dòng chữ |
| RLHF | **RLHF** | Con người xếp hạng câu trả lời → mô hình phần thưởng → AI học cư xử hữu ích |
| Energy | **Chi phí năng lượng** | Mỗi lượt nhỏ nhưng nhân triệu người thành lớn |
| **Reasoning** | **Chain-of-thought / reasoning models** | "Nghĩ" từng bước chính xác hơn "trả lời ngay" — xem 2.9 |
| Summary | **Tổng kết** | Ôn lại các ý cốt lõi |

## 2.9. Chain-of-thought & reasoning models (phòng mới — QUAN TRỌNG)

Đây là phòng 25 (`reasoning.js`) và là **điểm nhấn hiện đại** của dự án — bạn nên nắm rất kỹ.

**Chain-of-thought (CoT) là gì?** Là kỹ thuật cho mô hình ngôn ngữ **viết ra các bước suy luận trung gian** trước khi đưa đáp án cuối, thay vì trả lời ngay. Ví dụ với câu hỏi "có bao nhiêu chữ *r* trong *strawberry*": trả lời ngay dễ buột ra "2" (sai), nhưng nếu đánh vần từng chữ, đánh dấu, rồi đếm thì ra "3" (đúng).

**Vì sao "nghĩ" lại chính xác hơn?** Cốt lõi: mô hình sinh chữ bằng cách đoán từ tiếp theo (xem phòng Next-token). Khi bắt trả lời ngay, nó chỉ có "một phát đoán". Khi cho viết từng bước, **mỗi bước viết ra trở thành ngữ cảnh cho bước sau**, nên lời giải được *dựng dần* thay vì đoán một lần. Nói cách khác, model không "thông minh hơn" — nó chỉ có **thêm không gian để tính toán**.

**Reasoning models là gì?** Là các mô hình (như **o1** của OpenAI, **DeepSeek-R1**) được huấn luyện để **tự động** sinh một chuỗi suy nghĩ dài (thường ẩn) trước khi chốt câu trả lời — thay vì phải nhắc "hãy nghĩ từng bước". CoT là *kỹ thuật prompt*; reasoning model là *mô hình được huấn luyện để nội tại hoá kỹ thuật đó*.

**Đánh đổi:** "nghĩ" tốn **nhiều token hơn và chậm hơn** (phòng minh hoạ bằng huy hiệu token: trả lời ngay ≈ 1–3 token, nghĩ từng bước ≈ 48–70 token). Vì vậy chỉ nên "nghĩ kỹ" cho bài khó (toán, logic, lập trình nhiều bước); bài đơn giản thì trả lời ngay để tiết kiệm chi phí/độ trễ.

> 💡 **Mẹo phỏng vấn**: Nếu được hỏi "bạn có theo kịp xu hướng AI không?", hãy kể về phòng Reasoning: "Em thêm hẳn một phòng cho *reasoning models* — so sánh trực quan 'trả lời ngay' (hay sai) với 'suy nghĩ từng bước' (đúng hơn), kèm huy hiệu token để cho thấy đánh đổi tốc độ/chi phí ⟷ độ chính xác. Đó là ý tưởng đứng sau o1 và DeepSeek-R1."
