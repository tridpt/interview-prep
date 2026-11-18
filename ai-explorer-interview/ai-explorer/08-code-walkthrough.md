# 8. Code Walkthrough — đi qua luồng chạy thật

Phụ lục này đi qua **luồng chạy thật** của những phần cốt lõi, để bạn hiểu cặn kẽ 100% mã nguồn khi bị "soi sâu". Ba phần: (1) luồng khởi động + `route()`, (2) một file phòng điển hình, (3) phòng Reasoning mổ xẻ chi tiết.

---

## 8.1. Luồng khởi động → render phòng đầu tiên

1. `index.html` nạp `<script type="module" src="app.js">`. Vì là module, mọi `import` được phân giải trước.
2. Cuối `app.js`:
   ```js
   buildToolbar();   // dựng nút tìm/ngôn ngữ/âm thanh/trình chiếu
   initAnalytics();  // thống kê ẩn danh (tuỳ chọn)
   route();          // render theo hash hiện tại
   ```
3. `route()` đọc `currentRoute()` (`parseHash().id`). Nếu URL không có hash → `id = "home"`.
4. Với home: `renderProgress(null)` + `renderHome(app, ROOMS, navigate)` → vẽ hero, bộ chọn lộ trình, lưới thẻ phòng.
5. Song song, `pwa.js` đăng ký `sw.js`. Lần đầu: SW `install` precache `ASSETS`. Lần sau: phục vụ từ cache → mở tức thì cả khi offline.

## 8.2. `route()` khi mở một phòng (vd `#/reasoning`)

```js
function route() {
  const id = currentRoute();
  window.dispatchEvent(new CustomEvent("roomleave")); // (1) dọn phòng cũ
  app.innerHTML = "";
  window.scrollTo({ top: 0, behavior: "smooth" });
  applyTheme(id === "home" ? "home" : id);            // (2) đổi màu chủ đề
  const room0 = ROOMS.find((r) => r.id === id);
  trackView(id === "home" ? "/" : "/" + id, room0 ? tx(room0.title) : "AI Explorer");
  app.classList.remove("enter"); void app.offsetWidth; app.classList.add("enter"); // (3) animation

  if (id === "home") { renderProgress(null); renderHome(app, ROOMS, navigate); return; }

  const idx = ROOMS.findIndex((r) => r.id === id);
  if (idx === -1) { navigate("home"); return; }        // (4) hash lạ → về home

  const room = ROOMS[idx];
  visited.add(room.id); markVisited(room.id);           // (5) đánh dấu đã ghé
  renderProgress(room.id);

  // (6) khung sườn chung: tiêu đề + nút chia sẻ
  const head = document.createElement("div"); head.className = "room-head";
  head.innerHTML = `… ${tx(room.question)} … <h2>${tx(room.title)}</h2> …
    <button id="shareRoomBtn">${tx(UI.shareRoom)}</button>`;
  app.appendChild(head);
  head.querySelector("#shareRoomBtn").onclick = () => shareRoom(room);

  const body = document.createElement("div");
  app.appendChild(body);
  const renderRoom = await room.render();                // (7) tải module phòng theo nhu cầu
  if (generation !== routeGeneration || currentRoute() !== room.id) return;
  renderRoom(body);
  if (room.id !== "summary" && hasMicroQuiz(room.id)) renderMicroQuiz(body, room.id); // (8) quiz nhỏ
  showHint(room.id);                                     // (9) gợi ý onboarding

  // (10) nút Trước / Sau lấy từ ROOMS[idx-1] / ROOMS[idx+1]
  const prev = ROOMS[idx - 1], next = ROOMS[idx + 1];
  // …dựng nút…
}
```

Điểm mấu chốt cần nắm:
- **`roomleave`** phát *trước* khi xoá DOM → phòng cũ kịp dọn timer/webcam.
- Khung sườn (head, quiz, nav) do `app.js` lo → phòng chỉ viết nội dung.
- Hash không khớp phòng nào → tự quay về home (chống link hỏng).

## 8.3. Một file phòng điển hình (khuôn mẫu)

Mọi phòng theo cùng khuôn (ví dụ `neural-net.js`):

```js
import { sfx } from "../sound.js";
import { tx } from "../i18n.js";

export function roomNeuralNet(root) {
  root.innerHTML = `… <canvas id="nnCanvas"></canvas> …
    <div class="takeaway">${tx("💡 Điều cốt lõi: …", "💡 Key idea: …")}</div>`;

  const canvas = root.querySelector("#nnCanvas");
  let net = initNet(6), timer = null, epochs = 0;

  root.querySelector("#trainBtn").onclick = (e) => {
    if (timer) { clearInterval(timer); timer = null; return; }
    timer = setInterval(() => {
      for (let i = 0; i < 3; i++) { trainStep(net, data, 0.1); epochs++; }
      refresh(); // vẽ lại ranh giới + cập nhật độ chính xác
    }, 40);
  };

  window.addEventListener("roomleave", () => { if (timer) clearInterval(timer); }, { once: true });
}
```

Ba việc mọi phòng đều làm:
1. **Dựng HTML qua `tx`** (song ngữ), thường có `.panel`, `.row`, `.takeaway`.
2. **Gắn tương tác** (nút, thanh trượt, canvas).
3. **Dọn dẹp** ở `roomleave` (timer/animation/webcam) — bắt buộc vì không có framework.

`neural-net.js` tự cài lan truyền tiến (`forward`) và lan truyền ngược thủ công (`trainStep`) cho MLP 2→H→1, rồi vẽ vùng dự đoán lên canvas theo từng ô 7px — minh hoạ trực quan "AI đang học".

## 8.4. Phòng Reasoning (`reasoning.js`) — mổ xẻ chi tiết

Đây là phòng đáng để bạn thuộc lòng vì nó là điểm nhấn thời sự.

### Cấu trúc dữ liệu
```js
const PROBLEMS = [
  { id: "strawberry",
    q:    { vi: "Có bao nhiêu chữ 'r' trong 'strawberry'?", en: "…" },
    fast: { vi: "2", en: "2" },              // đáp án bốc đồng (SAI)
    fastNote: { vi: "Đoán nhanh theo cảm giác…", en: "…" },
    steps: [ /* các bước suy luận, song ngữ */ ],
    answer: { vi: "3", en: "3" },            // đáp án đúng
    fastTok: 1, thinkTok: 48,                // token minh hoạ
  },
  // penraser (bài toán bút–tẩy kiểu CRT), weekday (tính thứ trong tuần)…
];
```
Mỗi bài **deterministic**: đáp án sai, chuỗi bước, đáp án đúng, và số token đều cố định — nên demo luôn nhất quán và không cần gọi model thật.

### Khôi phục trạng thái từ deep-link
```js
const startP = parseInt(getParam("p", ""), 10);
let pIdx = Number.isInteger(startP) && PROBLEMS[startP] ? startP : 0;
```
Đọc tham số `p` từ URL (`#/reasoning?p=2`) để mở đúng bài đã chia sẻ; sai/không có thì về bài 0.

### Tôn trọng giảm chuyển động
```js
const reduce = window.matchMedia?.("(prefers-reduced-motion: reduce)").matches;
const stepDelay = reduce ? 0 : 700;
```
Nếu người dùng bật "giảm chuyển động", các bước hiện *ngay* thay vì nhỏ giọt.

### "Trả lời ngay" — bốc đồng, thường sai
```js
fastBtn.onclick = () => {
  const p = PROBLEMS[pIdx];
  fastBtn.disabled = true;
  fastOut.innerHTML = `
    <div class="rs-answer bad">${tx(UI.answerLabel)} <b>${tx(p.fast)}</b></div>
    <p class="muted rs-note">${tx(p.fastNote)}</p>
    ${tokBadge(p.fastTok)}`;   // huy hiệu ≈ 1–3 token
  sfx.wrong();
};
```

### "Suy nghĩ từng bước" — dựng dần rồi ra đáp án đúng
```js
thinkBtn.onclick = () => {
  const p = PROBLEMS[pIdx];
  clearTimers();
  thinkOut.innerHTML = `<ol class="rs-steps"></ol>`;
  const list = thinkOut.querySelector(".rs-steps");

  p.steps.forEach((s, si) => {
    const reveal = () => { /* thêm <li> bước si */ sfx.tick(); };
    if (stepDelay === 0) reveal();
    else timers.push(setTimeout(reveal, stepDelay * (si + 1)));  // nhỏ giọt từng bước
  });

  const finish = () => { /* thêm đáp án đúng .good + tokBadge(p.thinkTok) */ sfx.success(); };
  if (stepDelay === 0) finish();
  else timers.push(setTimeout(finish, stepDelay * (p.steps.length + 1)));
};
```

### Chọn bài & lưu deep-link
```js
function selectProblem(i) {
  pIdx = i;
  setParams({ p: i || null });   // ghi ?p=i vào URL (replaceState); bài 0 thì bỏ tham số
  clearTimers();                 // dọn timer bài cũ
  // reset hai cột output về trạng thái "chờ"…
}
```

### Dọn tài nguyên khi rời phòng
```js
const timers = [];
const clearTimers = () => { timers.forEach(clearTimeout); timers.length = 0; };
window.addEventListener("roomleave", clearTimers, { once: true });
```

### Điều cốt lõi phòng truyền tải (khối `.takeaway`)
> AI không "thông minh hơn" khi suy nghĩ từng bước — nó chỉ có **thêm không gian để tính**. Mỗi bước viết ra trở thành ngữ cảnh cho bước sau, nên lời giải dựng dần thay vì đoán một phát. Cái giá là **chậm hơn và tốn nhiều token hơn**, nên chỉ cần "nghĩ kỹ" cho bài khó — đúng điều các reasoning model làm tự động.

> 💡 Khi kể phòng này trong phỏng vấn, nhấn 3 ý: (1) **cơ chế** (mỗi bước là ngữ cảnh cho bước sau), (2) **đánh đổi** (token/độ trễ ⟷ độ chính xác, minh hoạ bằng huy hiệu token), (3) **khác biệt** CoT (kỹ thuật prompt) vs reasoning model (mô hình huấn luyện để tự nghĩ, như o1/DeepSeek-R1).

---

## 8.5. Kiểm tra nhanh trước khi vào phỏng vấn

Chạy thử để tự tin mọi thứ khớp:
```bash
npm run check-assets   # ✅ mọi asset runtime có trong ASSETS của sw.js
npm run test:i18n      # ✅ mọi {vi,en} đủ hai vế
npm run serve          # rồi mở http://localhost:8000 và bấm thử phòng Reasoning
```
