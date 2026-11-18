# 8. Phụ lục: Đi bộ qua code (Code Walkthrough) — đọc từng hàm thật

> ⚠️ File này được viết sau khi **đọc trực tiếp 100% `app.js` (2864 dòng)**, `sound.js`, `sw.js`, `index.html`, `style.css`. Đây là phần "cặn kẽ nhất" — đi qua **từng hàm thật, từng biến thật** theo đúng thứ tự trong code. Đọc kèm file `04-code-chi-tiet.md` (tổng quan) và mở `app.js` bên cạnh.

`app.js` là **một IIFE** `(() => { "use strict"; ... })()`. Dưới đây là toàn bộ nội dung bên trong, theo thứ tự.

---

## PHẦN 1 — I18N (đa ngôn ngữ)

```js
let LANG = localStorage.getItem("capcuu101_lang") || "vi";
function t(v) {
  if (v && typeof v === "object" && ("vi" in v || "en" in v)) return v[LANG] ?? v.vi;
  return v;
}
```
- `LANG`: biến toàn cục, đọc từ localStorage, mặc định `"vi"`.
- `t(v)` (translate): **trái tim của i18n**. Nếu `v` là object có `vi`/`en` → trả `v[LANG]`, fallback `v.vi` (toán tử `??` = nullish coalescing). Nếu `v` là chuỗi thường → trả nguyên. Nhờ vậy `t()` dùng được cho cả object song ngữ lẫn chuỗi đơn.
- `UI`: một object lớn `{ key: {vi, en} }` chứa **mọi chuỗi giao diện tĩnh** (~90 key: nút, tiêu đề, nhãn cài đặt, thông báo...). Ví dụ `play: { vi: "🎮 CHƠI & HỌC", en: "🎮 PLAY & LEARN" }`.

> 💡 Phỏng vấn: "Toàn bộ text song ngữ nằm tập trung ở đâu?" → object `UI` cho text tĩnh; còn nội dung (level/quiz/panic) thì song ngữ ngay trong cấu trúc dữ liệu.

---

## PHẦN 2 — NAVIGATION (router)

```js
const screens = {};
document.querySelectorAll(".screen").forEach(s => (screens[s.id] = s));
let current = "screen-home";
let emergStop = null, emergRender = null;   // gán bởi initEmergency
```
- `screens`: map `{ id: element }` của mọi `<section class="screen">`.
- `current`: id màn đang hiển thị.
- `emergStop`/`emergRender`: **tham chiếu hàm gán động** từ module `initEmergency` (vì module đó là IIFE riêng, `go()` cần gọi ngược vào nó).

**Hàm `go(id)` — bộ điều hướng:**
```js
function go(id) {
  if (!screens[id]) return;
  if (current === "screen-game") stopGame();        // dọn game cũ
  if (current === "screen-panic") stopPanic();       // dừng TTS + metronome
  if (current === "screen-emergency" && emergStop) emergStop();  // dừng máy đếm nhịp
  if (id === "screen-panic" && window.Sfx) Sfx.stopMusic();      // tắt nhạc khi vào Panic
  screens[current].classList.remove("active");
  screens[id].classList.add("active");
  screens[current].setAttribute("aria-hidden", "true");
  screens[id].removeAttribute("aria-hidden");
  current = id;
  // gọi render tương ứng:
  if (id === "screen-levels") renderLevels();
  if (id === "screen-achievements") renderAchievements();
  if (id === "screen-emergency" && emergRender) emergRender();
  if (id === "screen-handbook") renderHandbook();
}
```
➡️ **Điểm cốt lõi:** trước khi rời màn, `go()` **dọn dẹp tài nguyên** của màn cũ (game/panic/metronome) — chống rò rỉ. Sau đó đổi class `.active` (CSS lo việc ẩn/hiện) + `aria-hidden` (accessibility) + gọi đúng hàm render.

```js
document.querySelectorAll("[data-goto]").forEach(btn =>
  btn.addEventListener("click", () => go(btn.dataset.goto)));
```
- Mọi nút có `data-goto="screen-xxx"` tự động được gắn click → `go()`. Đây là cách "khai báo điều hướng trong HTML".

---

## PHẦN 3 — LƯU SAO (`capcuu101_stars`)

```js
function loadStars()  { try { return JSON.parse(localStorage.getItem(SAVE_KEY)) || {}; } catch { return {}; } }
function saveStars(id, stars) { const all = loadStars(); if (!all[id] || all[id] < stars) all[id] = stars; localStorage.setItem(...); }
function totalStars() { return Object.values(loadStars()).reduce((a,b)=>a+b, 0); }
```
- `loadStars()`: đọc `{ id: số_sao }`, **bọc try/catch** phòng JSON hỏng (mẫu lặp lại khắp file).
- `saveStars()`: **chỉ ghi đè nếu điểm mới cao hơn** (giữ kỷ lục).
- `totalStars()`: cộng tổng sao.

---

## PHẦN 4 — STATS, STREAK & BADGES (`capcuu101_stats`)

```js
function loadStats() {
  return Object.assign(
    { wins: {}, totalWins: 0, best: {}, streak: { count: 0, last: "" }, badges: [] },
    JSON.parse(localStorage.getItem(STATS_KEY)) || {} );
}
```
- `Object.assign(default, saved)` → đảm bảo luôn đủ các trường kể cả dữ liệu cũ thiếu (kỹ thuật **migration nhẹ**).

**Streak (chuỗi ngày) — logic tinh tế:**
```js
function todayStr()  { ... return "YYYY-M-D"; }
function daysBetween(a, b) { ... return số ngày giữa 2 chuỗi; }
function recordWin(id) {
  const s = loadStats();
  s.wins[id] = (s.wins[id]||0)+1; s.totalWins++;
  const t = todayStr();
  if (s.streak.last !== t) {                 // chưa tính hôm nay
    const gap = s.streak.last ? daysBetween(s.streak.last, t) : 999;
    s.streak.count = gap === 1 ? s.streak.count + 1 : 1;   // liền ngày → +1, ngắt → reset 1
    s.streak.last = t;
  }
  saveStats(s);
}
```
➡️ Nếu lần chơi cuối là **hôm qua** (`gap === 1`) → chuỗi +1; nếu cách xa hơn → reset về 1. Chơi nhiều lần cùng ngày không cộng thêm (vì `last === t`).

**`saveBest(id, val)`**: lưu kỷ lục combo CPR / điểm survival (chỉ ghi nếu cao hơn).

**`BADGES` — mảng 9 huy hiệu:** mỗi cái có `test(stats, stars)` trả về boolean. Ví dụ:
```js
{ id: "cpr_master", test: (s, st) => (st.cpr || 0) >= 3 }       // 3 sao màn CPR
{ id: "all_stars",  test: (s, st) => LEVELS.every(l => (st[l.id]||0) >= 3) }  // 3 sao mọi màn
{ id: "streak7",    test: (s) => s.streak.count >= 7 }          // chuỗi 7 ngày
```

**`checkBadges()`** — trả về danh sách huy hiệu **vừa** mở khoá:
```js
BADGES.forEach(b => { if (!s.badges.includes(b.id) && b.test(s, st)) { s.badges.push(b.id); newly.push(t(b.name)); } });
```
Chỉ thêm huy hiệu chưa có và `test` đúng → tránh báo trùng.

---

## PHẦN 5 — DỮ LIỆU MÀN CHƠI (`LEVELS`)

Mảng 17 object, mỗi object:
```js
{ id:"cpr", emoji:"🐻", type:"rhythm", char:"🐻",
  title:{vi,en}, desc:{vi,en}, lesson:{vi,en} }
```
- `type` ∈ `rhythm | thrust | hose | press | quiz` → quyết định engine nào chạy.
- `char`: emoji nhân vật hiển thị trong stage.
- `lesson`: bài học đời thực hiện ở màn kết quả.
- 5 màn thao tác (cpr/choke/infant_cpr→rhythm, burn→hose, bleed→press) + 12 màn `quiz`.

---

## PHẦN 6 — RENDER DANH SÁCH MÀN & THÀNH TÍCH

**`renderLevels()`**: cập nhật tổng sao, gọi `renderStreakBanner()`, rồi:
1. Tạo thẻ **Survival** đầu danh sách (có `role="button"`, `tabindex="0"`, nghe cả click lẫn Enter/Space → accessibility).
2. Lặp `LEVELS`, mỗi màn tạo `.level-card` hiện emoji + tiêu đề + sao đã đạt (`"★".repeat(got)+"☆".repeat(3-got)`), click → `startGame(lv)`.

**`renderStreakBanner()`**: tính chuỗi còn "sống" không (`daysBetween(last, today) <= 1`), hiện banner phù hợp (giữ phong độ / chơi để nối / chưa có chuỗi).

**`renderAchievements()`**: dựng HTML thống kê (tổng sao, màn 3 sao, chuỗi, lượt cứu, combo CPR, số huy hiệu) + lưới huy hiệu (mở khoá hiện emoji, chưa thì 🔒) + nút chia sẻ thành tích gọi `shareCard()`.

---

## PHẦN 7 — CẨM NANG (`renderHandbook`) & `formatStepText`

```js
function formatStepText(s) {
  return s.replace("một một năm","115").replace("năm centimet","5cm")
          .replace("bốn centimet","4cm").replace("đánh quận răng","đánh răng");
}
```
➡️ **Mẹo TTS quan trọng:** dữ liệu nguồn viết "một một năm" để máy đọc đúng "115"; khi **hiển thị** thì thay lại số. (Cùng logic dùng ở `renderPanicStep`.)

`renderHandbook()`: lặp `HANDBOOK_ORDER` (17 key theo thứ tự ưu tiên), lấy dữ liệu từ `PANIC_EN`/`PANIC`, dựng **accordion** (`.hb-item`): bấm `.hb-head` toggle class `open` + cập nhật `aria-expanded` + `Sfx.blip()`. **Tái dùng đúng dữ liệu PANIC** (DRY).

---

## PHẦN 8 — ENGINE GAME (khung chung)

```js
const stage = $("game-stage"), instr = $("game-instruction"), controls = $("game-controls");
let gameState = null;
const gameScreen = $("screen-game");
function clearOverlays() { gameScreen.querySelectorAll(".result-overlay").forEach(o => o.remove()); }
```

**`startGame(lv)`**:
```js
if (window.Sfx) { Sfx.unlock(); Sfx.startMusic(); }   // mở khoá audio (autoplay policy) + nhạc nền
go("screen-game"); clearOverlays();
// reset tiêu đề/HP/stage/controls
if (tutorialSeen(lv.type)) runGame(lv);
else showTutorial(lv, () => runGame(lv));   // lần đầu kiểu game này → hiện tutorial
```

**`runGame(lv)`**: `switch` theo `lv.type` → gọi `initRhythm/initThrust/initHose/initPress/initQuiz`.

**`stopGame()`**: `if (gameState && gameState.cleanup) gameState.cleanup(); gameState = null;` → **dọn dẹp**. Mỗi init phải đặt `gameState = { cleanup() {...} }`.

**`floatText(txt, color)`**: tạo `.float-text` bay lên rồi tự xoá sau 800ms (hiệu ứng "PERFECT!", "-40❤️"...).

---

## PHẦN 9 — SURVIVAL (`startSurvival` / `initSurvival`)

- `startSurvival()`: như startGame nhưng vào `initSurvival()`.
- `initSurvival()`:
  - Gom **mọi câu hỏi** từ `QUIZZES(_EN)` vào `pool`, xáo trộn (`sort(() => Math.random()-0.5)`).
  - `PER = 9000` ms/câu. Biến: `idx, score, combo, alive`.
  - `tickTimer()`: dùng `requestAnimationFrame`, tính thời gian còn lại, đổ thanh `tfill`; **3 giây cuối kêu tích tắc** (`Sfx.tick()` mỗi giây); hết giờ → `timeout()`.
  - `ask()`: hiện câu hỏi, render các đáp án (xáo trộn), mỗi nút `addEventListener(..., {once:true})`.
  - `choose()`: đúng → score++, combo++, hiệu ứng, câu tiếp; sai → tô đỏ + tô xanh đáp án đúng + `end(false)`.
  - `end(cleared)`: lưu `best.survival` (`saveBest`), `checkBadges()`, overlay kết quả + nút chia sẻ/chơi lại/về.
  - `gameState = { cleanup() { alive=false; cancelAnimationFrame(rafId); } }`.

---

## PHẦN 10 — TUTORIAL (`TUTORIALS`, `showTutorial`)

- `TUTORIALS`: `{ gameType: { emoji, title{}, text{} } }` — hướng dẫn cho 5 kiểu game.
- `tutorialSeen(type)` / `markTutorial(type)`: đọc/ghi `capcuu101_tut` (object `{ type: true }`).
- `showTutorial(lv, onStart)`: hiện overlay `#tutorial`, gắn handler nút bắt đầu **một lần** (`removeEventListener` ngay khi bấm), `markTutorial`, rồi gọi `onStart()`.

---

## PHẦN 11 — CHIA SẺ (`shareCard`, `buildShareImage`)

```js
function buildShareImage(bigText, subText) {
  const c = document.createElement("canvas"); c.width = c.height = 1080;
  // vẽ gradient nền, chữ thập đỏ (arc + 2 hình chữ nhật), tiêu đề, điểm, link
  return new Promise(res => c.toBlob(res, "image/png"));
}
async function shareCard(bigText, subText) {
  const blob = await buildShareImage(...);
  if (blob && navigator.canShare) { ... navigator.share({ files:[file], ... }); return; }
  if (navigator.share) { await navigator.share({ text, url, title }); return; }
  // fallback: clipboard
  await navigator.clipboard.writeText(text + "\n" + SHARE_URL); toast(...);
}
```
➡️ **Canvas vẽ thẻ ảnh** → **Web Share API** (ưu tiên chia sẻ kèm file ảnh; nếu không thì chia sẻ text; cuối cùng fallback chép link). Bắt `AbortError` (người dùng huỷ hộp thoại) để không báo lỗi.

`toast(msg)`: hiện thông báo nổi 2.2s.

---

## PHẦN 12 — `showResult(lv, success, stars, extra)`

- Nếu thắng: `saveStars` + `recordWin`; phát `Sfx.win()` (thua: `Sfx.logout()`) + rung.
- `checkBadges()` → nếu có huy hiệu mới hiện thông báo.
- **Chế độ nghiêm túc (`PREFS.serious`)**: đổi emoji 💀→⚠️ và đổi tiêu đề "ĐĂNG XUẤT KHỎI CUỘC ĐỜI" → "Chưa đạt — thử lại".
- Hiện bài học (`t(lv.lesson)`) + nút Chia sẻ/Chơi lại/Danh sách. Overlay gắn vào `#screen-game` (phủ toàn màn).

---

## PHẦN 13 — 5 ENGINE GAME (chi tiết)

### `initRhythm(lv)` — CPR theo nhịp
- `TARGET=20` nhịp đúng, `BPM=110`, `interval=60000/BPM`, `tolerance=230` ms.
- `setInterval(tick, interval)`: mỗi nhịp phát `Sfx.tick()` + rung + nảy vòng `ring`.
- `onTap()`: tính `phase = now - last`, `diff = min(phase, interval-phase)`. Nếu `diff <= tolerance` → đúng (combo++, "PERFECT!"); sai → báo "Nhanh quá/Chậm quá". Đạt `TARGET` → tính sao theo độ chính xác `acc` (>0.9→3 sao, >0.7→2, còn lại 1), `saveBest(lv.id, bestCombo)`, `showResult`.
- Chơi được bằng **Space/Enter** (accessibility).
- `cleanup() { clearInterval(beatTimer); }`.

### `initThrust(lv)` — Heimlich (cảm biến nghiêng)
- `NEED=6` lần đẩy mạnh. `doThrust(strong)`: đủ mạnh → thrusts++, đủ NEED → `win()`.
- **`useTiltMode()`**: nghe `deviceorientation`, đọc `e.beta`, chuẩn hoá về 0-100; vượt 65 (và `armed`) → 1 cú đẩy; <40 → `armed=true` lại (chống đếm liên tục).
- **`useTapMode()`**: fallback nhấn-giữ-thả; giữ >250ms = đẩy mạnh.
- **Logic chọn chế độ**: iOS (có `DOE.requestPermission`) → nút xin quyền; có DOE thường → tiltMode + **probe 2.5s**, nếu không nhận dữ liệu cảm biến → tự fallback tap; không có DOE → tap.
- `cleanup()`: gỡ listener `deviceorientation`.

### `initHose(lv)` — chữa bỏng
- `NEED=6500` ms làm mát. `place()`: đặt vết thương ngẫu nhiên trong stage.
- Pointer events (`pointerdown/move/up/leave`) di chuyển vòi nước; `pointerInside` kiểm tra có đè đúng vết thương không (so toạ độ với `getBoundingClientRect`).
- `setInterval(loop, 60)`: nếu đang xịt đúng → `cooled += dt`, **thỉnh thoảng vết bỏng "lan"** (`sinceFlare > 2000` → `place()` lại + `Sfx.wrong()`); nếu rời vòi → `cooled` giảm dần (nóng lại). Đủ `NEED` → thắng 3 sao.
- `cleanup()`: clearInterval + gỡ mọi pointer listener.

### `initPress(lv)` — cầm máu (2 bước)
- **Bước 1 — kéo gạc**: pointer drag gạc; `gup()` tính khoảng cách tâm gạc tới tâm vết thương (`Math.hypot`), `<55px` → đặt đúng → `startHold()`.
- **Bước 2 — giữ lực**: `startHold()` dựng thanh lực; `setInterval(60ms)`:
  - **Vùng xanh dao động**: `zLo = 52 + Math.sin(drift*0.9)*16` (36%-68%), `zHi = zLo+26`.
  - `force += (holding ? 0.18 : -0.22) * dt/16` (giữ thì tăng, thả thì giảm).
  - Nếu `force` trong [zLo, zHi] → `inZone += dt`; đủ `NEED_ZONE=3500` ms → thắng.
- `cleanup()`: clearInterval + gỡ listener.

### `initQuiz(lv)` — trắc nghiệm
- `qs = QUIZZES_EN[lv.id] || QUIZZES[lv.id]` (chọn theo ngôn ngữ). `hp=100`.
- `render()`: hết câu → tính sao theo HP còn lại (100→3, ≥60→2, còn lại 1) → thắng. Còn câu → hiện câu + đáp án **xáo trộn**, mỗi nút `{once:true}`.
- `pick()`: đúng → hiệu ứng + câu tiếp; sai → tô đỏ, **tô xanh đáp án đúng** (`item.options.find(o=>o.ok).t`), `hp -= 40`, nếu `hp<=0` → thua (hiện `wrongMsg`), còn HP → câu tiếp.
- `cleanup() {}` (quiz không có timer/listener cần dọn).

---

## PHẦN 14 — DỮ LIỆU NỘI DUNG: `QUIZZES`, `QUIZZES_EN`, `PANIC`, `PANIC_EN`

- `QUIZZES`/`QUIZZES_EN`: `{ id: [ { q, options:[{t, ok}], wrongMsg } ] }`. Mỗi câu **đúng 1 đáp án** `ok:true`. (Đây là loại dữ liệu **nguy hiểm nhất nếu sai** với app y tế — xem mục test ở file 06.)
- `PANIC`/`PANIC_EN`: `{ id: { title, metronome?, steps:[...] } }`. 17 tình huống. CPR/infant_cpr có `metronome:true`. Các bước viết "một một năm" để TTS đọc đúng.

---

## PHẦN 15 — PANIC MODE & TTS

**Gắn sự kiện:** mọi `.panic-card` click → `openPanic(c.dataset.emergency)`; các nút next/prev/replay/stop-voice/back-pick.

**`openPanic(key)`**: chọn dataset theo `LANG`, ẩn picker hiện steps, nếu `data.metronome` → hiện ring + `startMetronome()`, rồi `renderPanicStep()`.

**`renderPanicStep()`**: hiện `"{n}. " + text` (thay "một một năm"→"115"...), bật/tắt nút prev, đổi nhãn next thành "Xong" ở bước cuối, rồi `speakStep()`.

**`panicStep(dir)`**: tiến/lùi; vượt bước cuối → về Home.

**TTS:**
```js
let allVoices = [];
let chosenVoiceURI = localStorage.getItem("capcuu101_voice") || "";  // "" auto, "off" tắt
function refreshVoices() { allVoices = speechSynthesis.getVoices() || []; if (onVoicesReady) onVoicesReady(); }
speechSynthesis.onvoiceschanged = refreshVoices;   // getVoices() bất đồng bộ!
function hasViVoice() { return voicesForLang("vi").length > 0; }
function pickSpeakVoice() {
  if (chosenVoiceURI === "off") return "off";
  if (chosenVoiceURI) { tìm voice theo URI; }
  return voicesForLang(LANG==="en"?"en":"vi")[0] || null;   // auto
}
function speakStep() {
  speechSynthesis.cancel();              // huỷ câu đang đọc
  const v = pickSpeakVoice(); if (v === "off") return;
  const u = new SpeechSynthesisUtterance(raw);
  if (v) { u.voice=v; u.lang=v.lang; } else { u.lang = LANG==="en"?"en-US":"vi-VN"; }
  u.rate = 0.95; speechSynthesis.speak(u);
}
```
➡️ Xử lý đầy đủ: getVoices **async**, ưu tiên giọng đã chọn → giọng theo ngôn ngữ → tắt; `speechSynthesis.cancel()` trước khi nói câu mới (tránh chồng tiếng).

**Metronome Panic:** `startMetronome()` — `setInterval(600ms)` = 100 nhịp/phút, đếm 1-30, nảy ring + rung 40ms. `stopMetronome()` clearInterval.

**`stopPanic()`**: dừng metronome + `speechSynthesis.cancel()`.

---

## PHẦN 16 — NHẮC ÔN TẬP (Notifications)

- `reminderOn()`: đọc `capcuu101_reminder === "1"`.
- `setReminder(on)`: xin `Notification.requestPermission()`; được cấp → lưu + bắn thông báo xác nhận.
- `fireReminder(body)`: ưu tiên `serviceWorker.ready.showNotification` (đẹp hơn), fallback `new Notification`.
- `maybeRemind()`: khi mở app, nếu bật nhắc + có chuỗi + chưa chơi hôm nay → bắn nhắc. **Chỉ nhắc khi mở app** (không có push nền).

---

## PHẦN 17 — PREFERENCES (a11y + nghiêm túc)

```js
let PREFS = loadPrefs();   // { contrast, large, reduce, serious }
function applyPrefs() {
  document.body.classList.toggle("a11y-contrast", PREFS.contrast);
  document.body.classList.toggle("a11y-large", PREFS.large);
  document.body.classList.toggle("a11y-reduce", PREFS.reduce);
  document.body.classList.toggle("mode-serious", PREFS.serious);
}
```
➡️ Bật class trên `<body>`, CSS lo phần còn lại (đây là kiểu **"state ở JS, style ở CSS"** rất gọn).

---

## PHẦN 18 — SETTINGS MODAL (IIFE `initSettings`)

- Đồng bộ checkbox với `Sfx.settings` (sound/music/vibrate) và `reminderOn()`.
- Mỗi checkbox đổi → gọi `Sfx.setSound/setMusic/setVibrate` (phát thử tiếng/rung để xác nhận).
- Map `prefMap` nối 4 checkbox a11y với `PREFS` → `savePrefs()` + `applyPrefs()`.
- **Trình chọn giọng**: `populateVoiceSelect()` đổ "Tự động" + "Tắt" + giọng tiếng Việt trước + còn lại; `updateVoiceWarn()` hiện cảnh báo nếu thiếu giọng Việt; nút thử giọng → `speakSample()`. `onVoicesReady = populateVoiceSelect` để chạy lại khi giọng nạp xong (async).

---

## PHẦN 19 — EMERGENCY TOOLKIT (IIFE `initEmergency`)

- **Máy đếm nhịp CPR độc lập** (110 BPM): `startMetro/stopMetro` toggle `setInterval`, đếm 1-30, `Sfx.tick()`+rung. Gán `emergStop = stopMetro` để `go()` dừng được.
- **Thẻ y tế** (`capcuu101_medcard`):
  - `esc(s)`: **chống XSS** — `div.textContent = s; return div.innerHTML;` (escape ký tự đặc biệt).
  - `renderMC()`: dựng thẻ; số điện thoại → `tel:` sau khi **lọc còn `[\d+]`** (`replace(/[^\d+]/g,"")`).
  - `form submit`: `preventDefault`, gom giá trị `.trim()`, lưu localStorage, `toast`. Gán `emergRender = renderMC`.

---

## PHẦN 20 — APPLY LANGUAGE & INIT

**`applyLang()`**:
- Toggle `body.lang-en`, đặt `documentElement.lang`.
- Quét `[data-i18n]` đặt `textContent` từ `UI`.
- Cập nhật nhãn `.panic-card` từ `title` của dữ liệu PANIC.
- Cập nhật trạng thái nút `.lang-opt` (`active` + `aria-pressed`).
- Đặt `aria-label` cho nút icon (back/stop-voice/settings).

**`setLang(lang)`**: lưu localStorage + `applyLang()` + render lại màn đang mở.

**Khối INIT (chạy ngay):**
```js
applyLang(); applyPrefs();
// aria-hidden mọi màn trừ current
hideSplash();   // ẩn splash sau tối thiểu 1.4s
// đọc query: ?panic=1 → Panic; ?tools=1 → Tools; else Home
setTimeout(maybeRemind, 1800);
// đăng ký service worker khi 'load'
if ("serviceWorker" in navigator) window.addEventListener("load", () => navigator.serviceWorker.register("sw.js").catch(...));
```

---

## 🎯 Bảng tra cứu nhanh "hàm ↔ chức năng" (toàn bộ app.js)

| Nhóm | Hàm chính |
|------|-----------|
| i18n | `t`, `UI`, `applyLang`, `setLang` |
| Router | `go`, `screens`, `current` |
| Lưu trữ | `loadStars/saveStars/totalStars`, `loadStats/saveStats`, `recordWin`, `saveBest` |
| Streak | `todayStr`, `daysBetween`, `recordWin` |
| Huy hiệu | `BADGES`, `checkBadges` |
| Dữ liệu | `LEVELS`, `QUIZZES(_EN)`, `PANIC(_EN)`, `TUTORIALS`, `HANDBOOK_ORDER` |
| Render | `renderLevels`, `renderStreakBanner`, `renderAchievements`, `renderHandbook` |
| Engine | `startGame`, `runGame`, `stopGame`, `initRhythm/Thrust/Hose/Press/Quiz`, `showResult`, `floatText` |
| Survival | `startSurvival`, `initSurvival` |
| Tutorial | `tutorialSeen`, `markTutorial`, `showTutorial` |
| Chia sẻ | `buildShareImage`, `shareCard`, `toast` |
| Panic | `openPanic`, `renderPanicStep`, `panicStep`, `formatStepText`, `stopPanic` |
| TTS | `refreshVoices`, `voicesForLang`, `hasViVoice`, `pickSpeakVoice`, `speakStep`, `speakSample` |
| Metronome | `startMetronome`, `stopMetronome` (Panic) + `initEmergency` (độc lập) |
| Nhắc | `reminderOn`, `setReminder`, `fireReminder`, `maybeRemind` |
| Prefs | `loadPrefs`, `savePrefs`, `applyPrefs` |
| Modal | `initSettings` (IIFE) |
| Toolkit | `initEmergency` (IIFE): metronome + thẻ y tế (`esc`, `renderMC`) |
| Khởi động | khối INIT + đăng ký SW |

> 💡 Nắm được bảng này + đọc một lượt file 08 là bạn có thể **chỉ tay vào bất kỳ hàm nào trong `app.js` và giải thích** — đó là mức "cặn kẽ nhất" cho phỏng vấn.
