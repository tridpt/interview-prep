# 📚 Kho ôn phỏng vấn — Bản đồ & Lộ trình | Interview Prep — Map & Plan

> Đây là **điểm bắt đầu** của toàn bộ kho. Đọc file này trước để biết có gì và học theo thứ tự nào. (Song ngữ: Tiếng Việt trước, English below.)

**Hồ sơ dùng ngay khi phỏng vấn:** [Tổng hợp 12 ứng dụng, cách kể, demo và giới hạn cần nói đúng](HO_SO_PHONG_VAN_TOAN_BO_UNG_DUNG.md).

---

# 🇻🇳 TIẾNG VIỆT

Kho này gồm **2 phần**: tài liệu theo **từng dự án** (12 dự án) và 2 thư mục **kỹ năng/luyện tập chung** (không gắn dự án cụ thể). Tất cả **song ngữ Việt/Anh** (bản tiếng Anh nằm trong thư mục con `*-en` hoặc `en/`).

## Cấu trúc kho

```
interview-prep/
├── README.md                       ← BẠN ĐANG Ở ĐÂY (bản đồ + lộ trình)
├── HO_SO_PHONG_VAN_TOAN_BO_UNG_DUNG.md  # Hồ sơ tổng hợp cho nhà tuyển dụng
├── SO_SANH_DU_AN.md                # So sánh 12 dự án — gặp JD nào kể dự án nào
├── CAU_HOI_TONG_HOP.md             # 40 câu hỏi phỏng vấn tổng hợp xuyên dự án
├── LO_TRINH_ON_TAP.md              # Kế hoạch ôn theo số ngày (3/7/14 ngày)
│
│   ─── TÀI LIỆU THEO DỰ ÁN (mỗi bộ có bản VI + bản -en) ───
├── agent-talk-interview/           # Nhiều agent AI trò chuyện — FastAPI, LLM, streaming, orchestration
├── cap-cuu-101-interview/          # Game học sơ cứu (PWA) — vanilla JS, Web Audio/Speech, offline
├── ai-explorer-interview/          # Bảo tàng AI tương tác 26 phòng (PWA) — vanilla JS, ES modules, chain-of-thought
├── quan-quen-map-interview/        # Bản đồ quán ăn — Leaflet, IndexedDB, 47 test Playwright, CI/CD
├── totp-manager-interview/         # Quản lý mã 2FA — tự cài TOTP (RFC 6238), scrypt+Fernet, mật mã
├── word-checker-interview/         # Kiểm tra file Word — python-docx, Flask, regex, đóng gói .exe
├── outlook-mail-reader-interview/  # Đọc mail đa tài khoản — OAuth/MSAL, Graph/IMAP, đa luồng
├── art-shield-interview/           # Bảo vệ tranh khỏi AI — CLIP, PGD/EoT, adversarial, PyTorch
├── token-counter-interview/        # Đo token/credit AI — FastAPI, tiktoken, quét log local
├── voice-clone-web-interview/      # Clone giọng tiếng Việt — viXTTS, Web Audio, WAV encoder
├── handwriting-to-word-interview/  # Viết tay sang Word — canvas, handwriting API, LaTeX/KaTeX
├── red-team-lab-interview/         # Lab prompt injection — 7+5 màn giả lập, sandbox
│
│   ─── KỸ NĂNG & LUYỆN TẬP CHUNG (dùng cho mọi dự án) ───
├── KyNang_PhongVan/   + en/        # Giới thiệu bản thân, STAR, câu hỏi ngược, viết CV, mock interview
└── LuyenTap_KyThuat/  + en/        # JS · Python · Web/Real-time · System Design · Coding · Flashcard · CS · Mermaid
```

Mỗi bộ **theo dự án** có 9–10 file đánh số: `00` mục lục · `01` tổng quan · `02` khái niệm nền tảng · `03` kiến trúc · `04`(+`05`) chi tiết code · tính năng nổi bật · câu hỏi phỏng vấn · cheat sheet · (một số có) code-walkthrough đi qua từng hàm.

## Nên học theo thứ tự nào?

**Giai đoạn 1 — Hiểu dự án (quan trọng nhất).** Chọn **1–2 dự án mạnh nhất / hợp JD nhất**, với mỗi dự án đọc `00 → 08` theo thứ tự. Mục tiêu: **kể được dự án trong 2 phút** + giải thích được 2 quyết định kỹ thuật + nêu được hạn chế.

**Giai đoạn 2 — Nền tảng kỹ thuật.** Đọc `LuyenTap_KyThuat/` theo ngôn ngữ liên quan (JS nếu frontend/Node, Python nếu backend/AI) + Web/Real-time + CS nền tảng.

**Giai đoạn 3 — Luyện trình bày.** `KyNang_PhongVan/` (giới thiệu bản thân, STAR, câu hỏi ngược). Nếu có vòng coding/thiết kế: `LuyenTap_KyThuat/04` + `05`.

**Phút chót (trước giờ G).** Lướt `cheat-sheet` của dự án + `LuyenTap_KyThuat/06_FLASHCARD.md` + `08_SO_DO_MERMAID.md`.

## Gợi ý theo định hướng vị trí
- **Frontend / Fullstack**: cap-cuu-101, quan-quen-map, ai-explorer, handwriting-to-word + `LuyenTap_KyThuat/01` (JS) + `03` (Web/Real-time).
- **Backend / Python**: agent-talk, totp-manager, word-checker, outlook, token-counter + `LuyenTap_KyThuat/02` (Python).
- **Thiên về bảo mật**: totp-manager (mật mã), red-team-lab (mô phỏng prompt injection) + `07_CS_NEN_TANG` (phần mật mã).
- **Thiên về AI/LLM**: agent-talk (orchestration, streaming, tra cứu web bổ sung ngữ cảnh); red-team-lab cho giáo dục an toàn LLM, ai-explorer cho giải thích AI trực quan.
- **Thiên về AI/ML, deep learning, computer vision**: art-shield (adversarial, CLIP, PGD/EoT) + voice-clone-web (TTS, voice cloning).

## Mẹo cốt lõi
- **Hiểu, đừng học vẹt** — người phỏng vấn sẽ hỏi vặn "tại sao".
- **Nối nền tảng với dự án thật** ("em dùng async vì gọi LLM mất vài giây").
- **Có số liệu + thành thật về hạn chế** — ghi điểm hơn là giả vờ hoàn hảo.

---

# 🇬🇧 ENGLISH

For a concise interview dossier covering all 12 apps, see [the Vietnamese portfolio briefing](HO_SO_PHONG_VAN_TOAN_BO_UNG_DUNG.md); the per-project folders below include English versions.

This repo has **2 parts**: **per-project** docs (12 projects) and 2 **general skill/practice** folders (not tied to a specific project). Everything is **bilingual VI/EN** (the English version is in the `*-en` subfolder or `en/`).

## Structure

```
interview-prep/
├── README.md                       ← YOU ARE HERE (map + plan)
├── SO_SANH_DU_AN.md                # Compare the 12 projects — which to lead with per JD
├── CAU_HOI_TONG_HOP.md             # 40 cross-project interview questions
├── LO_TRINH_ON_TAP.md              # Study plan by days (3/7/14 days)
│
│   ─── PER-PROJECT DOCS (each set has a VI version + an -en version) ───
├── agent-talk-interview/           # Multiple AI agents talking — FastAPI, LLM, streaming, orchestration
├── cap-cuu-101-interview/          # First-aid learning game (PWA) — vanilla JS, Web Audio/Speech, offline
├── ai-explorer-interview/          # Interactive AI museum, 26 rooms (PWA) — vanilla JS, ES modules, chain-of-thought
├── quan-quen-map-interview/        # Food-spot map — Leaflet, IndexedDB, 47 Playwright tests, CI/CD
├── totp-manager-interview/         # 2FA code manager — self-built TOTP (RFC 6238), scrypt+Fernet, crypto
├── word-checker-interview/         # Word-file checker — python-docx, Flask, regex, .exe packaging
├── outlook-mail-reader-interview/  # Multi-account mail reader — OAuth/MSAL, Graph/IMAP, threading
├── art-shield-interview/           # Protect art from AI — CLIP, PGD/EoT, adversarial, PyTorch
├── token-counter-interview/        # AI token/credit monitor — FastAPI, tiktoken, local log scan
├── voice-clone-web-interview/      # Vietnamese voice cloning — viXTTS, Web Audio, WAV encoder
├── handwriting-to-word-interview/  # Handwriting to Word — canvas, handwriting API, LaTeX/KaTeX
├── red-team-lab-interview/         # Prompt-injection lab — 7+5 simulated levels, sandbox
│
│   ─── GENERAL SKILLS & PRACTICE (for any project) ───
├── KyNang_PhongVan/   + en/        # Self-intro, STAR, reverse questions, CV writing, mock interview
└── LuyenTap_KyThuat/  + en/        # JS · Python · Web/Real-time · System Design · Coding · Flashcards · CS · Mermaid
```

Each **per-project** set has 9–10 numbered files: `00` index · `01` overview · `02` foundational concepts · `03` architecture · `04`(+`05`) code detail · standout features · interview questions · cheat sheet · (some) a function-by-function code walkthrough.

## What order to study?

**Phase 1 — Understand the projects (most important).** Pick your **1–2 strongest / most JD-relevant** projects; for each, read `00 → 08` in order. Goal: **pitch a project in 2 minutes** + explain 2 technical decisions + name limitations.

**Phase 2 — Technical fundamentals.** Read `LuyenTap_KyThuat/en/` for the relevant language (JS for frontend/Node, Python for backend/AI) + Web/Real-time + CS fundamentals.

**Phase 3 — Presentation practice.** `KyNang_PhongVan/en/` (self-intro, STAR, reverse questions). For coding/design rounds: `LuyenTap_KyThuat/en/04` + `05`.

**Last minute.** Skim the project's `cheat-sheet` + `LuyenTap_KyThuat/en/06_FLASHCARDS.md` + `08_MERMAID_DIAGRAMS.md`.

## Suggestions by target role
- **Frontend / Fullstack**: cap-cuu-101, quan-quen-map, ai-explorer, handwriting-to-word + `LuyenTap_KyThuat/en/01` (JS) + `03` (Web/Real-time).
- **Backend / Python**: agent-talk, totp-manager, word-checker, outlook, token-counter + `LuyenTap_KyThuat/en/02` (Python).
- **Security-leaning**: totp-manager (crypto), red-team-lab (prompt-injection simulator) + `07_CS_FUNDAMENTALS` (crypto section).
- **AI/LLM-leaning**: agent-talk (orchestration, streaming, web-search grounding); red-team-lab for LLM safety education, ai-explorer for visual AI education.
- **AI/ML, deep learning, computer vision-leaning**: art-shield (adversarial, CLIP, PGD/EoT) + voice-clone-web (TTS, voice cloning).

## Core tips
- **Understand, don't memorize** — interviewers keep asking "why".
- **Connect fundamentals to a real project** ("I used async because the LLM call takes a few seconds").
- **Use numbers + be honest about limitations** — it scores more than faking perfection.
