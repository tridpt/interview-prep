# 02 — Foundational Concepts (from zero)

---

## 1. What is TTS (Text-to-Speech)?

Technology that turns **text → speech**. Input is a character string, output is an audio signal (a waveform). Modern TTS uses neural nets to produce natural, human-like voices.

## 2. What is voice cloning?

Standard TTS reads with one (or a few) fixed, pre-trained voices. **Voice cloning** reads in a **specific person's voice**, learned from an audio sample. Two levels:

- **Zero-shot cloning**: needs only a **very short** sample (a few seconds); the model "captures" the voice features instantly, no retraining. XTTS does this.
- **Fine-tune cloning**: further train the model on **minutes** of someone's voice → closer and more stable, but more effort/resources.

## 3. XTTS and viXTTS

- **XTTS-v2** (Coqui): a multilingual TTS model with zero-shot cloning; the architecture uses a **GPT-like** part generating audio tokens + a decoder turning them into a waveform.
- **viXTTS**: an XTTS-v2 fine-tune for **Vietnamese** (viVoice dataset), with an **extended tokenizer** for better Vietnamese. The project uses the `capleaf/viXTTS` repo.

## 4. Conditioning latents & speaker embedding

The core cloning concept in XTTS. Given a sample audio, the model extracts two things:
- **speaker embedding**: a vector capturing the speaker's "timbre".
- **gpt_cond_latent** (conditioning latents): an audio-context representation so the generator "mimics" the speaking style.

In code: `model.get_conditioning_latents(audio_path=...)` returns `(gpt_cond_latent, speaker_embedding)`. Then `model.inference(text, language, gpt_cond_latent, speaker_embedding, ...)` generates the waveform reading `text` in that voice.

> 🔑 Idea: separate "the content to read" (text) from "the timbre" (latents from the sample). Swap the sample = swap the voice, no retraining.

## 5. Digital audio: sample rate, PCM, WAV

- **Sample rate**: samples per second, e.g. 22050 Hz, 24000 Hz. Higher = more detail.
- **PCM** (Pulse-Code Modulation): how "raw" audio is stored — each sample is a number (16-bit = integer −32768..32767).
- **WAV**: a file format holding PCM plus a **44-byte header** describing it (sample rate, channels, bits/sample...).
- **Mono vs stereo**: 1 vs 2 channels. TTS datasets usually use **mono**.

> Why WAV not webm? The fine-tune pipeline (LJSpeech-style) needs raw, uncompressed WAV.

## 6. Web Audio API & browser recording

Browsers grant mic access via `navigator.mediaDevices.getUserMedia`. To get raw audio:
- Create an `AudioContext`, connect the mic to a **ScriptProcessor** (or AudioWorklet).
- Each frame (buffer) yields a `Float32Array` (values −1..1).
- Concatenate frames, **downsample** to the target rate, then **encode WAV** (write header + convert float → int16).

> ⚠️ Mic recording requires the page on `localhost` or HTTPS — a browser security rule.

## 7. Downsampling

Mics typically record at 44100/48000 Hz, but we want 22050 Hz. **Downsampling** averages/selects samples to lower the rate. The project averages source samples within each proportional "window" — simple but sufficient.

## 8. What is fine-tuning?

**Fine-tune** = train *further* an existing model on new data (here: your voice) to specialize it. Needs:
- **Dataset**: (audio file, transcript) pairs — **LJSpeech** format: `wavs/0001.wav|the sentence read|speaker1`.
- **Duration**: 10–60 minutes, each clip 2–15 seconds, clean recording.
- **GPU**: train on Colab (T4/L4/A100). The notebook can use **Whisper** to auto-generate transcripts if you only have audio.

## 9. LJSpeech metadata

`metadata.csv` is `|`-separated; each line: `wav_path | sentence_text | speaker_name`. A standard format for many TTS pipelines. The project regenerates this from the recorded wavs.

## 10. Lazy loading the model

viXTTS is heavy (hundreds of MB) and slow to load. **Lazy load**: load the model into memory only on the **first TTS call**, not at app startup → the server boots fast. The first synthesis is slow (load + possibly download weights), later ones are fast.

## 11. Pluggable engine

Designed for easy model swaps: define a common "interface" — any engine with `synthesize(text, reference_wav, language) -> wav_path` works. To try another model, write a new class and set `engine = YourEngine()`. A simple form of **dependency inversion**.

## 12. FastAPI: Form, File, UploadFile, StreamingResponse

- **`Form(...)`**: receives form data (not JSON), good with file upload.
- **`UploadFile` / `File(...)`**: receives uploaded files (audio).
- **`FileResponse`**: returns a file (the result wav).
- **`StreamingResponse`**: streams data (e.g. a zip built in memory).

---

## Quick glossary

| Term | One line |
|------|----------|
| TTS | Turn text into speech |
| Voice cloning | Read in a voice learned from a sample |
| Zero-shot | Clone from a few seconds, no retraining |
| XTTS / viXTTS | Multilingual TTS model / Vietnamese fine-tune |
| Conditioning latents | Voice features + context from the sample audio |
| Speaker embedding | Vector for the speaker's "timbre" |
| Sample rate | Audio samples per second (22050, 24000 Hz) |
| PCM / WAV | Raw audio / file with PCM + header |
| Downsample | Lower the sample rate |
| Fine-tune | Further train on new data |
| LJSpeech | Dataset format (wav|text|speaker) |
| Lazy load | Load the model only when needed |
| Pluggable engine | Easy model swap via a common interface |
