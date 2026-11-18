# 08 — Cheat Sheet (last-minute skim)

## 30-second pitch
> Voice Clone Web clones Vietnamese voices with viXTTS. Record/upload a few-second sample, type text, the model extracts conditioning latents then reads the text in that voice. FastAPI backend with a pluggable, lazy-loaded engine; recording encodes 16-bit WAV in the browser; a dataset studio + fine-tune notebook for higher quality. Mandatory consent for ethics.

## Three highlights
1. **Cloning via conditioning latents** (speaker embedding + gpt_cond_latent) → zero-shot.
2. **Hand-rolled client WAV encoder** (Web Audio → downsample → 44-byte header → int16).
3. **Pluggable engine + lazy-load + soft imports**.

## Endpoints
| Method | Path | Role |
|--------|------|------|
| GET | `/api/health` | device + recorded count |
| GET | `/api/sentences` | 200 sentences + recorded flag |
| POST | `/api/record` | save one clip |
| DELETE | `/api/record/{id}` | delete to re-record |
| GET | `/api/dataset/download` | zip dataset (BytesIO) |
| POST | `/api/tts` | generate voice (consent required) |

## TTS flow
consent? → validate text (≤1000) → _resolve_reference (upload > id > first) → engine.synthesize → get_conditioning_latents → inference(temperature=0.3, text_splitting) → save uuid.wav (24kHz) → FileResponse.

## synthesize (memorize)
`_load()` → `_normalize_vi()` → `get_conditioning_latents(sample)` → `inference(text, latents)` → `_save_wav(24000)`.

## One-line glossary
- **TTS**: text → speech.
- **Voice cloning**: read in a voice learned from a sample.
- **Zero-shot**: clone from a few seconds, no retraining.
- **viXTTS**: XTTS-v2 fine-tuned for Vietnamese.
- **Conditioning latents**: voice features + context from the sample.
- **PCM/WAV**: raw audio / file + 44-byte header.
- **Downsample**: lower the sample rate.
- **LJSpeech**: `wavs/x.wav|text|speaker`.
- **Lazy load**: load the model when needed.
- **Pluggable engine**: swap models via the `synthesize` interface.

## Numbers / params
- Recording sample rate: 22050 Hz; viXTTS output: 24000 Hz.
- MAX_TEXT_LENGTH = 1000; MIN_REFERENCE_SECONDS = 3.
- Fine-tune: 10–60 min (15–30 is enough), clips 2–15s.
- Repo: `capleaf/viXTTS` (Hugging Face).

## 44-byte WAV header
RIFF + size · WAVE · fmt (PCM=1, mono=1, sampleRate, byteRate=sr×2, blockAlign=2, bits=16) · data + size. Samples: float[-1,1] → int16 little-endian.

## Point-scorers
- Hand-rolled WAV encoder (real signal processing).
- Pluggable engine (decoupling), lazy + soft imports.
- Dataset → fine-tune pipeline (200 phoneme-covering sentences, Whisper transcripts).
- Consent gate on both UI and API.
- In-memory streaming zip.

## Go-to line on ethics
> "Voice cloning is easily abused for impersonation, so I added a mandatory consent step in both the UI and API; /api/tts refuses without consent. The project is clearly positioned as learning/research, only with your own voice or with consent."

## Common pitfalls
- Mic recording needs localhost/HTTPS.
- StaticFiles `/` mount goes **last**.
- Heavy imports (TTS/torch/vinorm) must be "soft" so the app boots.
- WAV not webm (pipeline needs raw PCM).
- CPU is slow → recommend GPU/Colab.
