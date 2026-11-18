# 📑 Index — Voice Clone Web (viXTTS) Interview Prep

Welcome! This set helps you deeply understand the **Vietnamese Voice Clone Web** project for interviews, even if you know nothing about TTS or audio processing. Read `00 → 08` in order.

> It's a **web app** that lets you **record/upload a voice sample** then **clone the voice to read Vietnamese text**. The default engine is **viXTTS** (XTTS-v2 fine-tuned for Vietnamese), cloning a voice from just a few seconds of audio. It also has a **dataset recording studio** and a **fine-tune notebook** for Colab.

## Reading order

| File | Title | Content | When |
|------|-------|---------|------|
| `00` | **Index** (this file) | Map of docs | First |
| `01` | [Project overview](01-project-overview.md) | What it is, the problem, main tech, ethics | Big picture |
| `02` | [Foundational concepts](02-foundational-concepts.md) | TTS, voice cloning, XTTS, voice embeddings, WAV/PCM, fine-tune — from zero | Theory base |
| `03` | [System architecture](03-system-architecture.md) | Backend/frontend, 2 tabs, data flow, pluggable engine | After concepts |
| `04` | [Backend deep dive](04-backend-deep-dive.md) | `main.py` (API), `tts_engine.py` (viXTTS), `config.py`, `sentences.py` | The core |
| `05` | [Frontend deep dive](05-frontend-deep-dive.md) | `app.js`, `wav-recorder.js` (record + WAV encode), 2 tabs | UI + recording |
| `06` | [Standout features](06-standout-features.md) | Point-scorers: lazy load, pluggable engine, client WAV encoder, dataset/fine-tune | Pick highlights |
| `07` | [Interview questions](07-interview-questions.md) | 60+ Q&A | Practice |
| `08` | [Cheat sheet](08-cheat-sheet.md) | One-page recap | Last minute |

## After reading you should be able to

1. Pitch the project in 2 minutes: what, for whom, with what tech.
2. Explain zero-shot vs fine-tune cloning, XTTS, conditioning latents.
3. Understand why recording encodes WAV **in the browser**, and the dataset → fine-tune pipeline.
4. Articulate the **ethics/legal** angle: only clone your own voice or with consent; the app has a mandatory consent step.

## One line to remember

> "Voice Clone Web clones Vietnamese voices with viXTTS: the user records/uploads a few-second sample, types text, the model extracts the voice features (conditioning latents) and reads the text in that voice. The engine is pluggable, the model loads lazily, audio is WAV-encoded in the browser, and there's a dataset studio + fine-tune notebook. A mandatory consent step exists for ethics."
