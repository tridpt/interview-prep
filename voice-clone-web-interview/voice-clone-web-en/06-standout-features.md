# 06 — Standout Features (your point-scorers)

---

## 1. Hand-rolled WAV encoder in the browser ⭐⭐⭐

**Why it stands out**: shows real **digital signal processing** knowledge — understanding the WAV structure and why it's needed.

**How to say it**: "I record the mic via Web Audio API, collect Float32 frames, downsample to 22050 Hz, then build the 44-byte WAV header by hand and convert each float [-1,1] sample to int16 little-endian. I use WAV, not webm, because the fine-tune pipeline needs raw PCM. This lets the backend just save the file — no ffmpeg/librosa."

## 2. Pluggable engine (decoupling backend ↔ model) ⭐⭐⭐

**Why it stands out**: an extensible design, a form of dependency inversion.

**How to say it**: "The backend just calls `engine.synthesize(text, reference_wav)`, unaware of the model inside. To switch to VieNeu-TTS or F5-TTS, I just write a class with the same method and reassign `engine`. This decoupling lets me experiment with new models without touching the backend."

## 3. Lazy-load + soft import (fast startup, resilient) ⭐⭐⭐

**How to say it**: "viXTTS weighs hundreds of MB, so I load it only on the first TTS call — the app boots instantly. Every heavy import (TTS, torch, vinorm) is wrapped in try/except: if a library is missing, the app still boots and reports a clear error exactly when needed, instead of crashing."

## 4. Voice cloning via conditioning latents ⭐⭐

**How to say it**: "viXTTS separates 'content' from 'timbre'. I call `get_conditioning_latents` on the sample to get the speaker embedding + gpt_cond_latent, then `inference` generates the waveform reading the text in that voice. Swapping the sample swaps the voice without retraining — that's zero-shot cloning."

## 5. Full dataset → fine-tune pipeline ⭐⭐

**How to say it**: "Beyond zero-shot, there's a studio recording 200 sample sentences designed to cover all tones/phonemes, auto-generating LJSpeech metadata, packaging dataset.zip, plus a Colab fine-tune notebook (using Whisper for transcripts). Users go from zero-shot to fine-tuned quality."

## 6. 200 purpose-built sample sentences ⭐

**How to say it**: "Fine-tune quality depends on phoneme coverage. I wrote 200 sentences spanning the 6 tones, hard consonants (khúc khuỷu, ngoằn ngoèo), numbers/dates, questions/exclamations — so the trained voice is natural and robust."

## 7. Vietnamese text normalization with fallback ⭐

**How to say it**: "I use vinorm to read numbers/symbols naturally (15% → fifteen percent), but vinorm often fails to build on Python 3.13, so I fall back softly: if missing, use the raw text and the engine still runs."

## 8. Mandatory consent — ethics by design ⭐⭐

**How to say it**: "Voice cloning is easily abused (impersonation, scams). I added a mandatory consent step in both the UI and API: `/api/tts` refuses without consent. The project is clearly positioned as learning/research, only with your own voice or with consent."

## 9. In-memory dataset packaging (streaming zip) ⭐

**How to say it**: "I zip the dataset into BytesIO and return a StreamingResponse — no temp zip on disk, clean and fast."

---

## "What to emphasize by JD" table

| JD emphasizes | Emphasize this |
|---------------|----------------|
| AI / ML / Speech | viXTTS, conditioning latents, zero-shot vs fine-tune, dataset |
| Backend (Python) | FastAPI, pluggable engine, lazy/soft import, streaming zip, consent gate |
| Frontend / Web | hand-rolled WAV encoder, Web Audio API, recording management, object URL |
| Signal/audio processing | WAV/PCM, downsample, sample rate, float→int16 |
| AI ethics / trust & safety | mandatory consent, clear positioning, misuse warning |
