# 07 — Interview Questions (with model answers)

---

## A. Overview (1–10)

**1. What does the project do in one sentence?**
A web app that clones Vietnamese voices: record/upload a few-second sample, then read any text in that voice, using viXTTS.

**2. The two main features?**
(1) Try zero-shot cloning; (2) a dataset studio + fine-tune for higher quality.

**3. What's the engine?**
viXTTS — an XTTS-v2 fine-tune for Vietnamese, run via Coqui TTS.

**4. Main technologies?**
FastAPI, viXTTS (Coqui TTS/PyTorch), huggingface_hub, torchaudio, vinorm (optional); vanilla JS frontend + Web Audio API.

**5. Who uses it?**
People wanting Vietnamese narration in their own voice — learning/research.

**6. Why viXTTS?**
Zero-shot cloning from ~6 seconds, and already fine-tuned for Vietnamese so it reads more naturally than base XTTS.

**7. Main endpoints?**
`/api/sentences`, `/api/record` (POST/DELETE), `/api/dataset/download`, `/api/tts`, `/api/health`.

**8. TTS input/output?**
In: text + voice sample (file or recorded) + consent. Out: a WAV reading the text in the cloned voice.

**9. The ethics concern?**
Cloning is easily abused; the app has mandatory consent, use only your own voice or with consent.

**10. Limitations?**
Zero-shot isn't perfect; CPU is slow (needs GPU/Colab); no auth if deployed publicly.

---

## B. Foundational concepts (11–28)

**11. What is TTS?**
Technology turning text into speech.

**12. How does voice cloning differ from regular TTS?**
Regular TTS uses a fixed voice; cloning reads in a voice learned from a person's sample.

**13. What is zero-shot cloning?**
Cloning from just a few seconds of audio, no retraining.

**14. How is fine-tune cloning different?**
Further training on minutes of voice → closer and more stable, but resource-heavy.

**15. XTTS-v2 and viXTTS?**
XTTS-v2: a multilingual zero-shot cloning TTS; viXTTS: a Vietnamese fine-tune with an extended tokenizer.

**16. What are conditioning latents?**
Features extracted from the sample audio to "mimic" the voice: speaker embedding + gpt_cond_latent.

**17. What is a speaker embedding?**
A vector describing the speaker's "timbre".

**18. What is sample rate?**
Audio samples taken per second (e.g. 22050, 24000 Hz).

**19. PCM and WAV?**
PCM: raw audio storage (each sample a number); WAV: a file with PCM + a 44-byte header.

**20. Why WAV not webm?**
The fine-tune pipeline needs raw uncompressed PCM; WAV is the TTS dataset standard.

**21. What is downsampling and why?**
Lowering the sample rate (e.g. 48000 → 22050) by averaging samples; because mics record higher than we need.

**22. What is fine-tuning?**
Further training an existing model on new data (your voice) to specialize it.

**23. The LJSpeech format?**
`metadata.csv` `|`-separated: `wavs/0001.wav|content|speaker1`.

**24. What is lazy load and its benefit?**
Load the model only on the first call → fast app startup.

**25. What is a pluggable engine?**
A design allowing easy model swaps via a common interface (`synthesize`).

**26. What's the Web Audio API used for here?**
Mic access, getting raw audio (Float32) to encode WAV.

**27. What is temperature in inference?**
The "randomness" of generation; low (0.3) → stable, clear voice.

**28. What does enable_text_splitting do?**
Auto-splits long text into chunks for steadier reading.

---

## C. Backend detail (29–46)

**29. Why does /api/record just save the file?**
Because the audio is already WAV-encoded on the client; the backend needs no heavy audio library.

**30. What does _rebuild_metadata do?**
Regenerates metadata.csv (LJSpeech) from current wavs to stay in sync.

**31. How are recorded files named?**
By sentence_id, 4 digits (`0001.wav`) → re-recording overwrites.

**32. How does /api/dataset/download build the zip?**
Zips into BytesIO in memory then StreamingResponse — no temp file on disk.

**33. What does /api/tts validate before synthesis?**
Consent (required), non-empty text, text ≤ MAX_TEXT_LENGTH (1000).

**34. How does _resolve_reference prioritize?**
Uploaded file > reference_id > first recorded sample; none → 400.

**35. Why import the engine inside /api/tts?**
Lazy — so the app isn't heavy at startup when TTS isn't needed.

**36. Why distinguish RuntimeError vs Exception in /api/tts?**
RuntimeError (TTS not installed) → 503; other errors → 500, with clear messages.

**37. What does _ensure_downloaded do?**
Checks weights; if missing, snapshot_download from Hugging Face.

**38. Why import TTS inside _load?**
So the app still boots without the TTS library; clear error only when truly needed.

**39. What are the steps in synthesize?**
load → normalize text → get_conditioning_latents(sample) → inference(text, latents) → save uuid WAV.

**40. Why save results with a uuid?**
To avoid filename collisions across generations.

**41. At what sample rate does _save_wav save?**
24000 Hz (viXTTS output), via torchaudio.

**42. _normalize_vi and its fallback?**
Uses vinorm to normalize numbers/symbols; if missing, uses the raw text (soft fallback).

**43. How does _detect_device work?**
Tries to import torch, returns "cuda" if a GPU exists, else "cpu"; missing torch also returns "cpu".

**44. What does config.py do at import?**
Creates model/uploads/outputs dirs, sets device, limit constants.

**45. What's special about sentences.py?**
200 sentences covering tones/phonemes/numbers/questions — important for fine-tune quality.

**46. What are the static mounts?**
`/storage` (replay audio) and `/` (frontend) — `/` goes last.

---

## D. Frontend & deep questions (47–62)

**47. How does WavRecorder encode WAV?**
Collect Float32 → downsample → build the 44-byte WAV header → convert float [-1,1] to int16 little-endian → Blob.

**48. Why ScriptProcessor?**
Simple, broadly compatible for collecting audio buffers (though AudioWorklet is more modern).

**49. What's in the 44-byte WAV header?**
RIFF/WAVE, fmt chunk (PCM=1, mono=1, sampleRate, byteRate, bitsPerSample=16), data chunk + size.

**50. How to convert float to int16?**
`s < 0 ? s*0x8000 : s*0x7fff` (clamp [-1,1] first) — map to the 16-bit integer range.

**51. Why record only one sentence at a time?**
To avoid multiple-recorder conflicts; `toggleRecord` blocks if another is recording.

**52. How is the TTS result played?**
`URL.createObjectURL(blob)` assigned to `<audio>` and the download button.

**53. Where is consent checked?**
Both frontend (alert/disabled button) and backend (/api/tts refuses without it).

**54. Why is CPU slow?**
The TTS model is large; waveform generation is compute-heavy. GPU (Colab) is much faster.

**55. Why is the first synthesis slow?**
It must load the model (lazy) and possibly download weights from Hugging Face.

**56. How to switch to another TTS model?**
Write a new class with `synthesize(text, reference_wav, language)` and set `engine = YourEngine()`.

**57. Security risk when deployed publicly?**
No auth yet; the API is open. The README notes auth is needed for real deployment.

**58. Next improvements?**
Auto-denoise the sample, better long-text splitting, add auth, a job queue for multiple users.

**59. Why separate content from timbre?**
For zero-shot cloning: swapping the sample swaps the voice without retraining.

**60. How much data for good fine-tuning?**
10–60 minutes (15–30 is already good), clips 2–15 seconds, clean, ≥22kHz.

**61. What is Whisper used for in fine-tuning?**
To auto-generate transcripts from audio if you only have audio without text.

**62. Biggest lesson from the project?**
Decoupling (pluggable/lazy) makes integrating heavy models clean; and voice cloning must come with ethical responsibility.

---

## Answering tips
- Emphasize three points: **cloning via conditioning latents**, **hand-rolled client WAV encoder**, **pluggable + lazy engine**.
- Always proactively mention **ethics/consent** — interviewers value responsibility.
- Be honest about limits (slow CPU, no auth) then offer improvements.
