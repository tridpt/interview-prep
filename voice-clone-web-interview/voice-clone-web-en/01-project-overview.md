# 01 — Project Overview

## 1. What is this project? (for a complete beginner)

**Text-to-Speech (TTS)** turns text into speech. **Voice cloning** is advanced TTS: instead of a fixed voice, the model learns a person's **voice features** from a short audio sample, then reads any text in *that* voice.

**Voice Clone Web** does this for **Vietnamese**:
1. The user **records** (via mic) or **uploads** a voice sample (a few seconds).
2. Types some Vietnamese text.
3. The model reads the text in the cloned voice, returning a WAV file to play/download.

It also has a **dataset recording studio**: read sample sentences to build a dataset, export `dataset.zip` to **fine-tune** (further train) for higher quality.

## 2. What problem does it solve?

| Problem | How it's handled |
|---------|------------------|
| Low-quality / robotic Vietnamese TTS | Use viXTTS — fine-tuned for Vietnamese from XTTS-v2 |
| Cloning usually needs lots of data | XTTS clones **zero-shot** from just ~6 seconds |
| Want an even closer voice | Dataset studio + Colab fine-tune notebook |
| Browser recording in the format the pipeline needs | Encode **16-bit PCM WAV** right in the browser |
| Misuse of cloning others' voices | A **mandatory consent** step before generating |

## 3. The engine: what is viXTTS?

- **XTTS-v2** (Coqui): a multilingual TTS model with strong zero-shot voice cloning.
- **viXTTS** (`capleaf/viXTTS` on Hugging Face): a fine-tune of XTTS-v2 on a Vietnamese dataset (viVoice), with an extended Vietnamese tokenizer → more natural Vietnamese.
- Runs via the **Coqui TTS** library (the maintained `coqui-tts` fork).

## 4. Main technologies

| Component | Tech | Role |
|-----------|------|------|
| Backend | **FastAPI** + `uvicorn` | Recording, dataset, TTS APIs |
| TTS model | **viXTTS** (Coqui TTS, PyTorch) | Clone voice + synthesize speech |
| Model download | **huggingface_hub** | Auto-download viXTTS weights |
| Audio (saving) | **torchaudio**, soundfile | Save the result WAV |
| Vietnamese text norm | **vinorm** (optional, with fallback) | Read numbers/symbols more naturally |
| Frontend | Vanilla HTML/CSS/JS | 2 tabs, recording, call API |
| Recording | **Web Audio API** (hand-rolled WAV encoder) | Mic → 16-bit PCM WAV |
| Fine-tune | Colab notebook (Whisper + train) | Train a custom voice |

## 5. Directory structure

```
voice-clone-web/
├── backend/
│   ├── main.py          # FastAPI: /api/sentences, /api/record, /api/dataset/download, /api/tts
│   ├── tts_engine.py    # viXTTS wrapper (pluggable, lazy-load)
│   ├── sentences.py     # 200 Vietnamese sample sentences for the dataset
│   └── config.py        # Paths, device, model, limits
├── frontend/
│   ├── index.html       # 2 tabs: Record data / Try cloned voice
│   ├── app.js           # Tabs, call API
│   ├── wav-recorder.js  # Record + WAV encode in the browser
│   └── style.css
├── finetune/            # Colab notebook to fine-tune viXTTS
└── storage/             # recorded dataset + generated audio
```

## 6. Two main features

1. **Try cloned voice (zero-shot)**: record/upload a sample → type text → hear it instantly. No training.
2. **Record dataset + fine-tune**: read sample sentences → download `dataset.zip` → fine-tune on Colab → bring the checkpoint back. Higher quality than zero-shot.

## 7. Ethics & legal (very important)

The project **emphasizes** this, and you should raise it proactively in interviews:
- Use only with **your own voice** or one with **explicit consent**.
- Cloning others' voices to impersonate/scam is **illegal**.
- The app has a **mandatory consent step** (a checkbox) on both the recording and generation tabs; without consent → the API refuses.
- Clearly positioned as **learning / research**.

## 8. What makes it notable

- **Pluggable engine**: to switch models (VieNeu-TTS, F5-TTS) just write a new class with the same `synthesize()` method.
- **Lazy-loaded model**: the app boots fast; the heavy model loads only on the first TTS call.
- **Client-side WAV encoder**: record mic, then downsample + encode 16-bit PCM mono WAV (not webm) because the fine-tune pipeline needs WAV.
- **200 carefully designed sample sentences**: covering the 6 tones, tricky consonants, numbers/dates, questions/exclamations — so the trained voice is natural.

## 9. 30-second pitch

> "Voice Clone Web clones Vietnamese voices with viXTTS. The user records or uploads a few-second sample, types text, and the model extracts conditioning latents from the sample then reads the text in that voice. The FastAPI backend uses a pluggable, lazy-loaded engine; recording encodes 16-bit WAV right in the browser; and there's a dataset studio + fine-tune notebook for higher quality. I added a mandatory consent step because of cloning's ethical concerns."
