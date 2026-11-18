# 🧭 So sánh 12 dự án — Chọn dự án nào để kể? | Project Comparison — Which to Lead With?

> Mục đích: khi gặp một JD (mô tả công việc) cụ thể, biết ngay **nên kể dự án nào trước** và **nhấn vào điểm kỹ thuật nào**. (Song ngữ: Tiếng Việt trước, English below.)

---

# 🇻🇳 TIẾNG VIỆT

## Bảng tổng quan nhanh

| Dự án | Một câu | Ngôn ngữ/Stack chính | Loại |
|-------|---------|----------------------|------|
| **agent-talk** | Nhiều agent AI tự trò chuyện theo thời gian thực | Python (FastAPI) + JS thuần | Backend + AI/LLM |
| **cap-cuu-101** | Game học sơ cứu (PWA), đọc to khi khẩn cấp | JS/HTML/CSS thuần (PWA) | Frontend thuần |
| **ai-explorer** | Bảo tàng AI tương tác 26 phòng, chạy trên trình duyệt | JS thuần + ES Modules (PWA) | Frontend + AI (giáo dục) |
| **quan-quen-map** | Bản đồ quán ăn theo gu cá nhân | JS thuần + Leaflet (PWA) | Frontend nâng cao |
| **totp-manager** | Quản lý mã 2FA, tự cài TOTP + mã hóa | Python (FastAPI) + JS thuần | Backend + Bảo mật |
| **word-checker** | Kiểm tra định dạng/chính tả file Word | Python (Flask/CLI) | Backend/Tooling |
| **outlook-mail-reader** | Đọc mail Outlook đa tài khoản | Python (FastAPI) + JS thuần | Backend + Tích hợp |
| **art-shield** | Demo nghiên cứu nhiễu đối kháng trên CLIP | Python (PyTorch/FastAPI) + JS thuần | AI/ML + Bảo mật |
| **token-counter** | Đo token/credit dùng AI từ log local | Python (FastAPI) + JS thuần | Backend + Data |
| **voice-clone-web** | Clone giọng nói tiếng Việt (TTS) | Python (FastAPI/viXTTS) + JS thuần | AI/Speech + Backend |
| **handwriting-to-word** | Viết tay thành văn bản Word, hỗ trợ công thức | JS (Vite, Canvas, Google Handwriting IME) | Frontend + Tích hợp |
| **red-team-lab** | Lab giáo dục prompt injection với engine giả lập | JS thuần (ES modules) | Frontend + Giáo dục bảo mật |

## Mỗi dự án thể hiện kỹ năng/khái niệm gì (điểm "ăn điểm")

| Kỹ năng / Khái niệm | agent-talk | cap-cuu-101 | quan-quen-map | totp-manager | word-checker | outlook |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| Python / FastAPI | ✅ | | | ✅ | (Flask) | ✅ |
| JS thuần (no framework) | ✅ | ✅ | ✅ | ✅ | ✅(web) | ✅ |
| PWA / offline / Service Worker | | ✅ | ✅ | ✅(nhẹ) | | |
| Streaming / async (SSE, async generator) | ✅✅ | | | | | ✅(threading) |
| LLM / AI / Orchestration / RAG | ✅✅ | | | | ✅(tùy chọn) | |
| Mật mã (KDF, AES, HMAC, RFC) | | | ✅(AES-GCM) | ✅✅ | | |
| Bản đồ / Geospatial (Leaflet) | | | ✅✅ | | | |
| Lưu trữ nâng cao (IndexedDB, 2 tầng) | | (localStorage) | ✅✅ | (vault file) | | |
| Web APIs trình duyệt (Audio/Speech/Canvas/Camera) | (TTS) | ✅✅ | ✅ | ✅(jsQR) | | |
| Tích hợp dịch vụ ngoài (OAuth/IMAP/Graph) | (LLM API) | | (OSM/OSRM) | | (LLM) | ✅✅ |
| Xử lý tài liệu (OOXML/python-docx) | | | | | ✅✅ | |
| Regex / xử lý văn bản | | | | (base32) | ✅✅ | |
| Test tự động + CI/CD | (pytest) | (thủ công) | ✅✅(47 Playwright+CI) | ✅✅(RFC vectors+property) | ✅(pytest) | |
| Đa luồng / concurrency | | | | | | ✅✅ |
| Đóng gói/phân phối (.exe, Docker) | (Docker) | (PWA tĩnh) | (GitHub Pages) | (PWA local) | ✅(.exe) | |
| Accessibility / i18n | | ✅(a11y+song ngữ) | ✅(song ngữ) | | | |

(✅✅ = điểm mạnh nổi bật của dự án đó)

### Hai dự án frontend mới

| Kỹ năng / Khái niệm | handwriting-to-word | red-team-lab |
|---|:---:|:---:|
| Canvas/Pointer Events, dựng `.docx` | ✅✅ | |
| Nhận diện bên ngoài / toán LaTeX | ✅✅ (Google IME; Gemini tùy chọn) | |
| Mô phỏng prompt injection / trust boundary | | ✅✅ (7 direct + 5 RAG) |
| Kiểm thử tự động | (chưa có) | ✅ (3 suite Node) |
| Giới hạn cần nói rõ | OCR cần mạng; toán Word là LaTeX text | Simulator không chứng minh jailbreak LLM thật |

### Ma trận bổ sung — AI/ML, dữ liệu và speech

| Kỹ năng / Khái niệm | art-shield | token-counter | voice-clone-web |
|---|:---:|:---:|:---:|
| Python / FastAPI | ✅ | ✅✅ | ✅ |
| JS thuần (no framework) | ✅ | ✅ | ✅ |
| Deep learning / PyTorch | ✅✅ | | ✅ |
| Adversarial ML (PGD, EoT, threat model) | ✅✅ | | |
| Computer vision / CLIP / embedding | ✅✅ | | |
| Đo lường (PSNR/SSIM, cosine, transfer) | ✅✅ | | |
| TTS / voice cloning / conditioning latents | | | ✅✅ |
| Xử lý tín hiệu/audio (WAV/PCM, downsample) | | | ✅✅ (WAV encoder client) |
| Token/Tokenizer/BPE (tiktoken) | | ✅✅ | |
| Xử lý log / data (regex, đệ quy, schema) | | ✅✅ | |
| Web APIs trình duyệt | | | ✅(Web Audio) |
| Tải/chạy model HF (huggingface_hub) | ✅(CLIP) | | ✅(viXTTS) |
| Engine pluggable / lazy load | | | ✅✅ |
| Đạo đức AI / consent / trung thực dữ liệu | ✅(giới hạn) | ✅✅(trung thực) | ✅✅(consent) |
| Đóng gói/phân phối | (Docker-able) | (.bat) | (Colab/ngrok) |

## Chọn dự án theo JD

| JD nhấn vào... | Kể trước | Nhấn điểm gì |
|----------------|----------|--------------|
| **AI / LLM / chatbot** | agent-talk | orchestration nhiều lượt, streaming, tra cứu web bổ sung ngữ cảnh, ánh xạ vai trò |
| **AI/ML, deep learning, computer vision** | art-shield | nhiễu đối kháng PGD/EoT tự cài, CLIP/embedding, gradient, PSNR/SSIM, transferability |
| **AI / Speech / TTS** | voice-clone-web | viXTTS, voice cloning zero-shot, conditioning latents, WAV encoder, fine-tune |
| **Backend / API (Python)** | agent-talk / totp-manager / word-checker | FastAPI, async, thiết kế tách lớp, REST |
| **Bảo mật / mật mã** | totp-manager | tự cài TOTP đúng RFC, scrypt+Fernet "không lưu khóa", test vector |
| **Frontend / PWA / offline** | quan-quen-map / cap-cuu-101 / ai-explorer | service worker, cache strategies, ES modules, IndexedDB, Web Audio/Speech |
| **AI education / giải thích AI trực quan** | ai-explorer | 26 phòng tương tác dạy khái niệm AI (kể cả chain-of-thought), kiến trúc module hoá, offline |
| **Bản đồ / dữ liệu vị trí** | quan-quen-map | Leaflet, markercluster/heatmap, OSM/OSRM, geospatial |
| **Test / chất lượng / CI-CD** | quan-quen-map / totp-manager | 47 test Playwright + CI; RFC vectors + property-based (hypothesis) |
| **Tích hợp hệ thống / OAuth** | outlook-mail-reader | MSAL device-code, Graph + fallback IMAP, refresh token xoay vòng, job đa luồng |
| **Xử lý tài liệu / tooling** | word-checker | OOXML "font hiệu lực", kiến trúc mở rộng, giảm false positive |
| **Canvas / OCR / xuất Word** | handwriting-to-word | nét vector, chia dòng, Google IME, toán LaTeX; nói rõ phụ thuộc mạng |
| **An toàn LLM / giáo dục bảo mật** | red-team-lab | 7+5 màn mô phỏng, trust boundary, test engine; không coi là benchmark LLM thật |
| **Data / log processing** | token-counter | quét log phòng thủ, dò usage đệ quy đa schema, chuẩn hóa, tiktoken |
| **Fullstack** | totp-manager / agent-talk | tự làm cả backend lẫn frontend + test + bảo mật |

## Nếu chỉ kịp chuẩn bị KỸ 2 dự án
- **Hồ sơ thiên backend/bảo mật**: **totp-manager** + **agent-talk**.
- **Hồ sơ thiên frontend**: **quan-quen-map** + **cap-cuu-101**.
- **Hồ sơ thiên AI/ML, nghiên cứu**: **art-shield** (adversarial, CLIP, PGD/EoT) + **voice-clone-web** (TTS, voice cloning) — hoặc ghép **art-shield** + **agent-talk** nếu JD nghiêng cả LLM.
- **Muốn an toàn, đa năng nhất**: **quan-quen-map** (test+CI+frontend) + **totp-manager** (backend+bảo mật) — phủ rộng nhất.

> 💡 Mẹo: mở đầu hãy nói "Em có vài dự án, nhưng liên quan nhất tới vị trí này là [X] vì [lý do khớp JD]" — cho thấy bạn đọc JD và biết chọn lọc.

---

# 🇬🇧 ENGLISH

## Quick overview

| Project | One line | Main language/stack | Type |
|---------|----------|---------------------|------|
| **agent-talk** | Multiple AI agents talk in real time | Python (FastAPI) + vanilla JS | Backend + AI/LLM |
| **cap-cuu-101** | First-aid learning game (PWA), reads aloud in emergencies | Vanilla JS/HTML/CSS (PWA) | Pure frontend |
| **ai-explorer** | Interactive AI museum, 26 browser rooms | Vanilla JS + ES Modules (PWA) | Frontend + AI (education) |
| **quan-quen-map** | A food-spot map by personal taste | Vanilla JS + Leaflet (PWA) | Advanced frontend |
| **totp-manager** | A 2FA manager, self-built TOTP + encryption | Python (FastAPI) + vanilla JS | Backend + Security |
| **word-checker** | A Word formatting/spelling checker | Python (Flask/CLI) | Backend/Tooling |
| **outlook-mail-reader** | A multi-account Outlook mail reader | Python (FastAPI) + vanilla JS | Backend + Integration |
| **art-shield** | Research demo of CLIP-targeted adversarial noise | Python (PyTorch/FastAPI) + vanilla JS | AI/ML + Security |
| **token-counter** | Measure AI token/credit usage from local logs | Python (FastAPI) + vanilla JS | Backend + Data |
| **voice-clone-web** | Vietnamese voice cloning (TTS) | Python (FastAPI/viXTTS) + vanilla JS | AI/Speech + Backend |
| **handwriting-to-word** | Handwriting to Word with math support | JS (Vite, Canvas, Google Handwriting IME) | Frontend + Integration |
| **red-team-lab** | Prompt-injection learning lab with simulated engine | Vanilla JS (ES modules) | Frontend + Security education |

## What each project demonstrates (the point-scorers)

| Skill / Concept | agent-talk | cap-cuu-101 | quan-quen-map | totp-manager | word-checker | outlook |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| Python / FastAPI | ✅ | | | ✅ | (Flask) | ✅ |
| Vanilla JS (no framework) | ✅ | ✅ | ✅ | ✅ | ✅(web) | ✅ |
| PWA / offline / Service Worker | | ✅ | ✅ | ✅(light) | | |
| Streaming / async (SSE, async generator) | ✅✅ | | | | | ✅(threading) |
| LLM / AI / Orchestration / RAG | ✅✅ | | | | ✅(optional) | |
| Cryptography (KDF, AES, HMAC, RFC) | | | ✅(AES-GCM) | ✅✅ | | |
| Maps / Geospatial (Leaflet) | | | ✅✅ | | | |
| Advanced storage (IndexedDB, two-tier) | | (localStorage) | ✅✅ | (vault file) | | |
| Browser Web APIs (Audio/Speech/Canvas/Camera) | (TTS) | ✅✅ | ✅ | ✅(jsQR) | | |
| External integration (OAuth/IMAP/Graph) | (LLM API) | | (OSM/OSRM) | | (LLM) | ✅✅ |
| Document processing (OOXML/python-docx) | | | | | ✅✅ | |
| Regex / text processing | | | | (base32) | ✅✅ | |
| Automated tests + CI/CD | (pytest) | (manual) | ✅✅(47 Playwright+CI) | ✅✅(RFC vectors+property) | ✅(pytest) | |
| Threading / concurrency | | | | | | ✅✅ |
| Packaging/distribution (.exe, Docker) | (Docker) | (static PWA) | (GitHub Pages) | (local PWA) | ✅(.exe) | |
| Accessibility / i18n | | ✅(a11y+bilingual) | ✅(bilingual) | | | |

(✅✅ = a standout strength of that project)

### Two newer frontend projects

| Skill / Concept | handwriting-to-word | red-team-lab |
|---|:---:|:---:|
| Canvas/Pointer Events and `.docx` generation | ✅✅ | |
| External recognition / LaTeX math | ✅✅ (Google IME; optional Gemini) | |
| Prompt-injection/trust-boundary simulation | | ✅✅ (7 direct + 5 RAG) |
| Automated tests | (none yet) | ✅ (3 Node suites) |
| Limit to disclose | OCR needs network; Word math is LaTeX text | Simulator does not prove real LLM jailbreaks |

### Supplementary matrix — AI/ML, data and speech projects

| Skill / Concept | art-shield | token-counter | voice-clone-web |
|---|:---:|:---:|:---:|
| Python / FastAPI | ✅ | ✅✅ | ✅ |
| Vanilla JS (no framework) | ✅ | ✅ | ✅ |
| Deep learning / PyTorch | ✅✅ | | ✅ |
| Adversarial ML (PGD, EoT, threat model) | ✅✅ | | |
| Computer vision / CLIP / embeddings | ✅✅ | | |
| Metrics (PSNR/SSIM, cosine, transfer) | ✅✅ | | |
| TTS / voice cloning / conditioning latents | | | ✅✅ |
| Signal/audio processing (WAV/PCM, downsample) | | | ✅✅ (client WAV encoder) |
| Token/Tokenizer/BPE (tiktoken) | | ✅✅ | |
| Log/data processing (regex, recursion, schema) | | ✅✅ | |
| Browser Web APIs | | | ✅(Web Audio) |
| Loading/running HF models (huggingface_hub) | ✅(CLIP) | | ✅(viXTTS) |
| Pluggable engine / lazy load | | | ✅✅ |
| AI ethics / consent / data honesty | ✅(limits) | ✅✅(honesty) | ✅✅(consent) |
| Packaging/distribution | (Docker-able) | (.bat) | (Colab/ngrok) |

## Pick a project by the JD

| The JD emphasizes... | Lead with | What to highlight |
|----------------------|-----------|-------------------|
| **AI / LLM / chatbot** | agent-talk | multi-turn orchestration, streaming, web-search grounding, role mapping |
| **AI/ML, deep learning, computer vision** | art-shield | hand-rolled PGD/EoT adversarial noise, CLIP/embeddings, gradients, PSNR/SSIM, transferability |
| **AI / Speech / TTS** | voice-clone-web | viXTTS, zero-shot voice cloning, conditioning latents, WAV encoder, fine-tune |
| **Backend / API (Python)** | agent-talk / totp-manager / word-checker | FastAPI, async, layered design, REST |
| **Security / cryptography** | totp-manager | self-built RFC-correct TOTP, scrypt+Fernet "no stored key", test vectors |
| **Frontend / PWA / offline** | quan-quen-map / cap-cuu-101 / ai-explorer | service worker, cache strategies, ES modules, IndexedDB, Web Audio/Speech |
| **AI education / explaining AI visually** | ai-explorer | 26 interactive rooms teaching AI concepts (incl. chain-of-thought), modular architecture, offline |
| **Maps / location data** | quan-quen-map | Leaflet, markercluster/heatmap, OSM/OSRM, geospatial |
| **Testing / quality / CI-CD** | quan-quen-map / totp-manager | 47 Playwright tests + CI; RFC vectors + property-based (hypothesis) |
| **System integration / OAuth** | outlook-mail-reader | MSAL device-code, Graph + IMAP fallback, rotating refresh tokens, multi-threaded job |
| **Document processing / tooling** | word-checker | OOXML "effective font", extensible architecture, false-positive reduction |
| **Canvas / OCR / Word export** | handwriting-to-word | vector strokes, line segmentation, Google IME, LaTeX; disclose network dependency |
| **LLM safety / security education** | red-team-lab | 7+5 simulated levels, trust boundaries, engine tests; not a real-LLM benchmark |
| **Data / log processing** | token-counter | defensive log scanning, recursive multi-schema usage detection, normalization, tiktoken |
| **Fullstack** | totp-manager / agent-talk | built both backend and frontend + tests + security |

## If you only have time to prep 2 projects deeply
- **Backend/security-leaning profile**: **totp-manager** + **agent-talk**.
- **Frontend-leaning profile**: **quan-quen-map** + **cap-cuu-101**.
- **AI/ML, research-leaning profile**: **art-shield** (adversarial, CLIP, PGD/EoT) + **voice-clone-web** (TTS, voice cloning) — or pair **art-shield** + **agent-talk** if the JD also leans LLM.
- **Safest, most versatile**: **quan-quen-map** (tests+CI+frontend) + **totp-manager** (backend+security) — widest coverage.

> 💡 Tip: open with "I have a few projects, but the most relevant to this role is [X] because [reason matching the JD]" — it shows you read the JD and can prioritize.
