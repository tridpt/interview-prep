# 5. Frontend chi tiết (giải thích `app.js`)

Frontend là **vanilla JavaScript** (không framework), nằm gần hết trong `frontend/app.js`. Nó là "bộ não" điều khiển vòng lặp hội thoại ở phía client.

---

## 5.1. Các biến trạng thái toàn cục

```javascript
let providers = [];        // danh sách provider lấy từ /api/config
let running = false;       // đang chạy vòng lặp hay không
let history = [];          // [{name, text, kind, sources}] — toàn bộ hội thoại
let lastSpeaker = "";      // người nói gần nhất
let summary = "";          // bản tóm tắt các lượt cũ
let summarizedCount = 0;   // đã tóm tắt tới lượt thứ mấy
```

Hằng số quan trọng:
- `SUMMARY_WINDOW = 16`: quá ngưỡng này thì bắt đầu tóm tắt.
- `KEEP_RECENT = 8`: luôn giữ nguyên văn 8 lượt gần nhất.
- `SS_KEY = "agentTalkSession"`: khóa lưu phiên trong localStorage.
- `LB_KEY = "agentTalkLeaderboard"`: khóa lưu bảng xếp hạng.

---

## 5.2. Khởi tạo — `init()`

```javascript
async function init() {
  await loadProviders();              // lấy provider từ backend
  const fromShare = await loadFromShare();   // ưu tiên mở link chia sẻ
  if (!fromShare && !restoreSession())       // nếu không thì khôi phục phiên cũ
    DEFAULT_AGENTS.forEach(addAgentCard);    // nếu vẫn không → 2 agent mặc định
  bindEvents();
  refreshUi();
  renderLeaderboard();
}
```
Thứ tự ưu tiên nguồn dữ liệu: **link chia sẻ → phiên localStorage → mặc định**.

---

## 5.3. Quản lý agent card — `addAgentCard()`, `getAgents()`

- `addAgentCard()`: nhân bản template HTML một thẻ agent, đổ provider vào dropdown, đổ persona mẫu vào dropdown, gắn sự kiện (chọn mẫu, xóa, lưu khi gõ). Mỗi agent có một **màu** lấy theo chỉ số (`COLORS`).
- `getAgents()`: đọc ngược từ DOM ra mảng object `{name, persona, provider, goal, memory}` để gửi lên server.
- Ràng buộc: tối thiểu 2 agent (chặn xóa nếu còn 2).

---

## 5.4. Vòng lặp chính — `loop()` (học kỹ phần này)

```javascript
async function loop(firstSpeaker) {
  running = true;
  const maxTurns = parseInt($("maxTurns").value) || 12;
  let speaker = firstSpeaker;
  for (let i = 0; i < maxTurns && running; i++) {
    await maybeSummarize();                       // nén ngữ cảnh nếu dài
    const target = newBubble(speaker);            // tạo bong bóng rỗng
    const { text, sources } = await streamTurn(speaker, target);  // đổ chữ dần
    if (!text) { ...; break; }                    // không có nội dung → dừng
    renderSources(bubbleEl, sources);             // hiện footnote nguồn
    history.push({ name: speaker, text, kind: "agent", sources });
    saveSession();
    tts.speak(speaker, text);                     // đọc to nếu bật
    speaker = await pickNext(speaker);            // chọn người nói tiếp
    if (delayMs) await sleep(delayMs);            // nghỉ giữa các lượt
  }
  await updateMemories();                         // cập nhật bộ nhớ cuối phiên
  stop();
}
```

**Điểm cần nắm:**
- Vòng lặp dừng khi: hết `maxTurns`, người dùng bấm Dừng (`running=false`), hoặc agent trả về rỗng.
- `running` được kiểm tra liên tục để **dừng giữa chừng** mượt mà (kể cả đang stream — xem `streamTurn`).

### `streamTurn()` — Đọc luồng từ server
```javascript
const resp = await fetch("/api/turn_stream", {... body: JSON.stringify({...})});
const reader = resp.body.getReader();
const dec = new TextDecoder();
let full = "";
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  full += dec.decode(value, { stream: true });   // ghép chunk
  target.textContent = full;                      // hiện dần
  scrollChat();
  if (!running) { reader.cancel(); break; }       // dừng giữa chừng
}
```
- Đọc `X-Sources` từ header để lấy nguồn web.
- `decodeURIComponent` để giải mã header (server đã `quote`).

### `pickNext()` — Chọn người nói tiếp
- Chế độ `auto` + >2 agent → gọi `/api/next_speaker` (LLM quyết).
- Ngược lại (hoặc lỗi) → **luân phiên**: người kế tiếp trong danh sách.

---

## 5.5. Chế độ tranh biện — `debateLoop()`

Khác `loop()` ở chỗ chạy theo **3 giai đoạn cố định** cho mọi agent:
```javascript
const DEBATE_PHASES = [
  { key: "mo_man",   label: "Mở màn" },
  { key: "phan_bien", label: "Phản biện" },
  { key: "ket_luan", label: "Kết luận" },
];
```
- Mỗi giai đoạn: hiện banner → từng agent nói (truyền `phase.key` để backend đổi prompt).
- Tranh biện xong → **tự động chấm điểm** (`scoreConversation()`).

---

## 5.6. Tóm tắt phía client — `maybeSummarize()`

```javascript
if (history.length - summarizedCount <= SUMMARY_WINDOW) return;  // chưa cần
const upTo = history.length - KEEP_RECENT;       // tóm tới đâu (giữ 8 lượt cuối)
const chunk = history.slice(summarizedCount, upTo);
// gọi /api/summarize với prev_summary + chunk
summary = j.summary; summarizedCount = upTo;
```
Sau khi tóm tắt, `streamTurn` chỉ gửi `history.slice(summarizedCount)` + `summary` → giữ payload nhỏ.

---

## 5.7. Đạo diễn — `interject()`

Người dùng gõ vào ô đạo diễn → đẩy một lượt `{name: "Đạo diễn", kind: "director"}` vào history. Lượt này sẽ xuất hiện trong lịch sử gửi lên server ở lượt sau (vai `user`), nên agent "nghe thấy" và phản ứng theo.

---

## 5.8. Lưu & khôi phục phiên (localStorage)

- `sessionData()`: gom toàn bộ trạng thái (agents, cấu hình, history, summary...) thành object.
- `saveSession()`: ghi vào localStorage (gọi liên tục mỗi khi có thay đổi).
- `restoreSession()`: đọc lại khi mở trang.
- `forgetSession()`: xóa phiên đã lưu.

---

## 5.9. Chia sẻ link — `shareLink()` & `loadFromShare()`

- `shareLink()`: gửi phiên lên `/api/session/save`, nhận mã, tạo URL dạng `...#s=<mã>`, chép vào clipboard.
  - **Lưu ý bảo mật/quyền riêng tư:** `getAgents().map(({ memory, ...rest }) => rest)` → **loại bỏ trường `memory`** trước khi chia sẻ (bộ nhớ riêng tư không gửi đi).
- `loadFromShare()`: đọc hash URL.
  - `#s=<mã>` → link ngắn, fetch từ server.
  - `#share=<base64>` → link cũ (nhúng cả phiên vào URL).
  - Sau khi nạp → `replaceState` xóa hash để reload không bị nạp lại.

---

## 5.10. Chấm điểm & bảng xếp hạng

- `scoreConversation()` → gọi `/api/score` → `renderScore()` vẽ thanh điểm từng tiêu chí + người thắng (👑).
- `recordScore()` → cộng dồn vào localStorage (`LB_KEY`): số phiên, số lần thắng, tổng điểm.
- `renderLeaderboard()` → bảng xếp hạng sắp theo số thắng, rồi điểm TB. Có huy chương 🥇🥈🥉.

---

## 5.11. Text-to-Speech — đối tượng `tts`

- Kiểm tra `"speechSynthesis" in window` (không phải trình duyệt nào cũng hỗ trợ).
- `voiceFor(name)`: chọn giọng **ổn định theo tên** bằng hàm băm đơn giản (cùng tên luôn cùng giọng), ưu tiên giọng tiếng Việt.

---

## 5.12. Bảo mật phía client: chống XSS — `escapeHtml()`

```javascript
const escapeHtml = (s) => s.replace(/[&<>"']/g, (c) => ({...}[c]));
```
Mọi nội dung do người dùng/LLM tạo (tên agent, lời thoại, tiêu đề nguồn) **đều được escape** trước khi nhét vào `innerHTML` → chống tấn công **XSS** (chèn mã độc qua nội dung). Link nguồn còn được kiểm tra `^https?://` mới cho bấm.

> 💡 **Điểm ăn điểm phỏng vấn:** Nêu được rằng bạn ý thức về XSS và đã escape nội dung động là một điểm cộng lớn về tư duy bảo mật.

---

## 5.13. Tóm tắt trách nhiệm frontend

| Nhóm hàm | Việc |
|----------|------|
| `init`, `loadProviders` | Khởi tạo |
| `addAgentCard`, `getAgents`, `recolor` | Quản lý agent |
| `loop`, `debateLoop`, `streamTurn`, `pickNext`, `cont`, `stop` | Vòng lặp hội thoại |
| `maybeSummarize` | Nén ngữ cảnh |
| `interject` | Đạo diễn xen vào |
| `saveSession`, `restoreSession`, `shareLink`, `loadFromShare` | Lưu/khôi phục/chia sẻ |
| `scoreConversation`, `renderScore`, `recordScore`, `renderLeaderboard` | Chấm điểm & xếp hạng |
| `updateMemories` | Bộ nhớ dài hạn |
| `save`, `copyConversation`, `buildContent` | Xuất file/chép |
| `tts.*` | Đọc to |
| `escapeHtml`, `renderSources` | Hiển thị an toàn |
