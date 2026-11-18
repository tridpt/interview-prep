# 05 — Chi tiết Frontend (đọc từng phần)

Frontend HTML/CSS/JS thuần, gồm `index.html`, `app.js`, `wav-recorder.js`, `style.css`. Phần đáng giá nhất là **WAV recorder** tự viết.

---

## A. `index.html` — bố cục 2 tab

- **Tabs**: 2 nút "1. Thu âm dữ liệu" và "2. Thử giọng clone".
- **Tab Studio** (`#studio`):
  - Card tiến độ: `progressText` (đã thu/tổng) + thanh `progressFill` + nút tải dataset.
  - Checkbox **consent** bắt buộc trước khi thu.
  - `#sentenceList`: render động 200 câu, mỗi câu có nút "Thu", audio nghe lại, nút "Xóa".
- **Tab Test** (`#test`):
  - Card nguồn giọng: tải file mẫu (tùy chọn) hoặc dùng mẫu đã thu.
  - Ô `textarea` nhập text (maxlength 1000) + bộ đếm ký tự.
  - Checkbox **consent** + nút "Tạo giọng nói".
  - Card kết quả (ẩn): `<audio>` phát + nút tải về.

## B. `wav-recorder.js` — class `WavRecorder` (⭐ điểm nhấn)

Tự thu micro và **encode WAV 16-bit PCM mono** ngay trên trình duyệt.

### `start()`
```js
this.mediaStream = await navigator.mediaDevices.getUserMedia({
  audio: { channelCount: 1, echoCancellation: true, noiseSuppression: true }});
this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
this.source = this.audioContext.createMediaStreamSource(this.mediaStream);
this.processor = this.audioContext.createScriptProcessor(4096, 1, 1);
this.processor.onaudioprocess = (e) => {
  if (this.recording) this.chunks.push(new Float32Array(e.inputBuffer.getChannelData(0)));
};
this.source.connect(this.processor);
this.processor.connect(this.audioContext.destination);
```
- Xin micro 1 kênh, bật khử vọng/khử ồn.
- Dùng **ScriptProcessor** (đơn giản, tương thích rộng) gom từng khung `Float32Array`.

### `stop()`
```js
this.recording = false;
const sampleRate = this.audioContext.sampleRate;   // vd 44100/48000
// ngắt kết nối, dừng track, đóng context
const merged = this._merge(this.chunks);
const down = this._downsample(merged, sampleRate, this.targetSampleRate);  // 22050
return this._encodeWav(down, this.targetSampleRate);   // Blob WAV
```

### `_merge(chunks)`
Nối các `Float32Array` rời rạc thành một mảng dài (tính tổng độ dài, copy lần lượt).

### `_downsample(buffer, inRate, outRate)`
Giảm sample rate (vd 48000 → 22050): với mỗi mẫu đầu ra, lấy **trung bình** các mẫu nguồn trong cửa sổ tương ứng. Nếu `outRate >= inRate` thì trả nguyên (không upsample).

### `_encodeWav(samples, sampleRate)` — ghi WAV thủ công
```js
const buffer = new ArrayBuffer(44 + samples.length * 2);  // 44 byte header + 16-bit
const view = new DataView(buffer);
// Ghi header RIFF/WAVE/fmt /data: format=PCM(1), mono(1), bitsPerSample=16...
view.setUint32(24, sampleRate, true);
view.setUint32(28, sampleRate * 2, true);   // byte rate
// Ghi dữ liệu: float [-1,1] → int16
for (...) view.setInt16(offset, s < 0 ? s * 0x8000 : s * 0x7fff, true);
return new Blob([view], { type: "audio/wav" });
```
- Tự dựng **header WAV 44 byte** + chuyển từng mẫu float [-1,1] sang **int16** (little-endian).
- Kết quả: Blob WAV chuẩn — gửi thẳng lên `/api/record`.
> Đây là phần "khó" và đáng kể nhất của frontend: hiểu cấu trúc file WAV và xử lý tín hiệu cơ bản.

## C. `app.js` — logic 2 tab

### Chuyển tab
Gắn click cho `.tab`: bỏ `active`/ẩn mọi panel, rồi kích hoạt tab + panel tương ứng.

### Tab Studio
- `loadSentences()`: `GET /api/sentences`, render danh sách, cập nhật tiến độ.
- `renderSentences()`: dựng từng câu (badge số, text, nút Thu/Xóa, audio); câu đã thu thì hiện audio + nút Xóa.
- `toggleRecord(id, btn)`:
  - Kiểm **consent** trước (chưa tích → alert).
  - Đang thu câu này → `stop()` lấy blob → `uploadSample`.
  - Đang thu câu khác → chặn (chỉ thu 1 lúc 1 câu).
  - Chưa thu → `new WavRecorder(22050)` + `start()`, đổi nút thành "■ Dừng & lưu".
- `uploadSample(id, blob)`: `POST /api/record` (FormData), cập nhật cờ recorded + render lại.
- `deleteSample(id)`: `DELETE /api/record/{id}`.
- `updateProgress()`: cập nhật text + bề rộng thanh tiến độ + bật/tắt nút tải dataset.
- Nút tải: `window.location.href = "/api/dataset/download"`.

### Tab Test
- Chọn file giọng mẫu → lưu `refBlob`.
- `textarea` input → cập nhật bộ đếm + `updateGenState()` (chỉ bật nút khi có text + consent).
- Nút "Tạo giọng nói":
  ```js
  const form = new FormData();
  form.append("text", testText.value.trim());
  form.append("consent", consentTest.checked);
  if (refBlob) form.append("audio", refBlob, refBlob.name || "ref.wav");
  const res = await fetch("/api/tts", { method: "POST", body: form });
  const blob = await res.blob();
  resultPlayer.src = URL.createObjectURL(blob);   // phát ngay
  ```
  - Hiện trạng thái "Đang xử lý... (lần đầu tải model, hơi lâu)".
  - Nhận blob WAV, tạo object URL để `<audio>` phát và cho tải.
  - Bắt lỗi → hiện message từ `detail` server trả về.

## D. `style.css`
- Theme tối, layout card, 2 tab, thanh tiến độ (`progressFill` đổi width).
- Câu đã thu tô nền xanh (`.done`); nút đang thu đổi đỏ (`.recording`).
- Responsive cơ bản.

## E. Điểm đáng nói khi phỏng vấn về frontend
1. **Tự viết WAV encoder**: hiểu cấu trúc file WAV (header 44 byte), chuyển float→int16, downsample — kiến thức xử lý tín hiệu thực sự.
2. **Đẩy xử lý nặng sang client**: backend chỉ lưu file, nhẹ và đơn giản.
3. **Quản lý trạng thái thu âm**: chỉ thu 1 câu/lúc, đổi nhãn nút, cập nhật tiến độ realtime.
4. **Consent gate ở cả 2 tab**: tôn trọng đạo đức ngay trên UI.
5. **Object URL để phát/tải kết quả**: dùng `URL.createObjectURL(blob)` cho audio sinh ra.
6. **Không framework**: thuần DOM + fetch + Web Audio API.
