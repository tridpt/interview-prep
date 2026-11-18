# 05 — Frontend Deep Dive (part by part)

Vanilla HTML/CSS/JS: `index.html`, `app.js`, `wav-recorder.js`, `style.css`. The most valuable part is the hand-rolled **WAV recorder**.

---

## A. `index.html` — 2-tab layout

- **Tabs**: two buttons "1. Record data" and "2. Try cloned voice".
- **Studio tab** (`#studio`):
  - Progress card: `progressText` (recorded/total) + `progressFill` bar + download dataset button.
  - A mandatory **consent** checkbox before recording.
  - `#sentenceList`: dynamically renders 200 sentences, each with a "Record" button, replay audio, and "Delete".
- **Test tab** (`#test`):
  - Voice-source card: upload a sample (optional) or use a recorded sample.
  - A `textarea` for text (maxlength 1000) + a char counter.
  - A **consent** checkbox + "Generate" button.
  - A result card (hidden): `<audio>` player + download button.

## B. `wav-recorder.js` — the `WavRecorder` class (⭐ highlight)

Records the mic and **encodes 16-bit PCM mono WAV** right in the browser.

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
- Request a 1-channel mic, enable echo/noise suppression.
- Use a **ScriptProcessor** (simple, broadly compatible) to collect each `Float32Array` frame.

### `stop()`
```js
this.recording = false;
const sampleRate = this.audioContext.sampleRate;   // e.g. 44100/48000
// disconnect, stop tracks, close context
const merged = this._merge(this.chunks);
const down = this._downsample(merged, sampleRate, this.targetSampleRate);  // 22050
return this._encodeWav(down, this.targetSampleRate);   // WAV Blob
```

### `_merge(chunks)`
Concatenates the separate `Float32Array`s into one long array (sum lengths, copy in order).

### `_downsample(buffer, inRate, outRate)`
Lowers the sample rate (e.g. 48000 → 22050): for each output sample, **average** the source samples in the corresponding window. If `outRate >= inRate`, return as-is (no upsampling).

### `_encodeWav(samples, sampleRate)` — write WAV by hand
```js
const buffer = new ArrayBuffer(44 + samples.length * 2);  // 44-byte header + 16-bit
const view = new DataView(buffer);
// Write RIFF/WAVE/fmt /data header: format=PCM(1), mono(1), bitsPerSample=16...
view.setUint32(24, sampleRate, true);
view.setUint32(28, sampleRate * 2, true);   // byte rate
// Write data: float [-1,1] → int16
for (...) view.setInt16(offset, s < 0 ? s * 0x8000 : s * 0x7fff, true);
return new Blob([view], { type: "audio/wav" });
```
- Builds the **44-byte WAV header** by hand + converts each float [-1,1] sample to **int16** (little-endian).
- Result: a standard WAV Blob — sent straight to `/api/record`.
> This is the hardest, most notable frontend part: understanding the WAV file structure and basic signal processing.

## C. `app.js` — 2-tab logic

### Tab switching
Click handler on `.tab`: remove `active`/hide all panels, then activate the corresponding tab + panel.

### Studio tab
- `loadSentences()`: `GET /api/sentences`, render the list, update progress.
- `renderSentences()`: build each sentence (number badge, text, Record/Delete buttons, audio); recorded ones show audio + Delete.
- `toggleRecord(id, btn)`:
  - Check **consent** first (unchecked → alert).
  - Recording this sentence → `stop()` to get the blob → `uploadSample`.
  - Recording another → block (only one at a time).
  - Not recording → `new WavRecorder(22050)` + `start()`, change the button to "■ Stop & save".
- `uploadSample(id, blob)`: `POST /api/record` (FormData), update the recorded flag + re-render.
- `deleteSample(id)`: `DELETE /api/record/{id}`.
- `updateProgress()`: update text + progress-bar width + toggle the download button.
- Download button: `window.location.href = "/api/dataset/download"`.

### Test tab
- Select a voice sample file → store `refBlob`.
- `textarea` input → update the counter + `updateGenState()` (enable the button only with text + consent).
- "Generate" button:
  ```js
  const form = new FormData();
  form.append("text", testText.value.trim());
  form.append("consent", consentTest.checked);
  if (refBlob) form.append("audio", refBlob, refBlob.name || "ref.wav");
  const res = await fetch("/api/tts", { method: "POST", body: form });
  const blob = await res.blob();
  resultPlayer.src = URL.createObjectURL(blob);   // play immediately
  ```
  - Shows "Processing... (first run loads the model, takes a while)".
  - Receives the WAV blob, creates an object URL for `<audio>` to play and download.
  - On error → shows the server's `detail` message.

## D. `style.css`
- Dark theme, card layout, 2 tabs, progress bar (`progressFill` width changes).
- Recorded sentences get a green background (`.done`); the active record button turns red (`.recording`).
- Basic responsiveness.

## E. Frontend talking points
1. **Hand-rolled WAV encoder**: understanding the WAV structure (44-byte header), float→int16 conversion, downsampling — real signal-processing knowledge.
2. **Push heavy processing to the client**: the backend just saves files, lean and simple.
3. **Recording state management**: only one sentence at a time, button label changes, real-time progress.
4. **Consent gate on both tabs**: ethics respected right in the UI.
5. **Object URL to play/download the result**: `URL.createObjectURL(blob)` for the generated audio.
6. **No framework**: plain DOM + fetch + Web Audio API.
