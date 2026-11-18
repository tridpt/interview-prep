# 8. Cheat Sheet — Quick Review Before Go-Time

Skim for 5-10 minutes before the interview.

---

## 🎯 30-second pitch (memorize)
> Word Checker: an **offline** tool reviewing `.docx` files before submission — checks **formatting** (font/size/spacing/alignment/margins), **mechanical text errors**, **spelling** (dictionary + optional AI), **headings/numbering**, **word/page limits**. Core: **Python + python-docx**, layered architecture (one module per check, errors are a shared `Issue`). Used via a **CLI** or a **Flask web app**; has **auto-fix**, **inline comments**, **learn-from-sample**, HTML/PDF/JSON export, **`.exe`** packaging (PyInstaller). Has **pytest + CI**.

## 🧱 Tech stack
- **Core**: Python 3.10+, python-docx, `re` (regex), argparse, dataclasses, Counter.
- **Interfaces**: Flask (web), CLI; **fpdf2** (PDF); **httpx + dotenv** (optional AI).
- **Packaging/test**: PyInstaller (.exe), pytest, GitHub Actions.

## 🗂️ Structure
cli.py / webapp.py / launch.py (interfaces) · config.py (profiles) · **checker/** (core).
checker: document (read+effective font) · issues (Issue) · runner (orchestration) · formatting / textcheck / spelling / llm_spelling / headings / limits / stats / mathdetect / learn_profile / autofix / annotate / report / pdf_report / resources.

## 🧩 Architecture (KNOW IT)
CLI + Web → **runner.run_checks(path, profile, do_*)** → modules return uniform **[Issue]** → report/pdf/json + autofix/annotate. Add a check = a `check_xxx(doc)->[Issue]` function + wire into the runner.

## 📄 Document reading (.docx = OOXML = zip of XML)
**Run** = same-formatting chunk; font/size at the run level. **Effective font** (⭐): run → character style → paragraph style (based_on) → docDefaults (read raw XML via qn()). `number` = index of NON-empty paragraphs.

## 🔎 The checks (regex)
double_space `  +` · space_before_punct `\s+([,.;:!?)])` · missing_space `([,;:])(?=\S)` (skip number:number) · repeated_words (backref, function words only) · capitalization (block abbrev/ordinal/ellipsis) · double_hyphen · mixed_quotes · placeholder · empty_paragraphs.

## 🚫 False-positive reduction (PHILOSOPHY)
Skip **math formulas** (mathdetect: symbols/eq-number/token-ratio≥35%) · "3,14"/"10:30" not flagged · "xa xa" not flagged as repeat · "Mục 1." not flagged for capitalization · alignment skips <12-word paragraphs · spelling only known pairs · **dedicated anti-false-positive tests**.

## ✍️ Spelling
Dictionary `common_typos.txt` (joined regex, longest first, IGNORECASE). AI (optional): OpenAI-compatible, chunking, temperature=0, **anti-hallucination** (phrase must exist), warning severity, lazy import.

## 📐 Profiles & learn-from-sample
config.PROFILES: hanh_chinh/luan_van/cong_ty + tolerances. **--template**: learn_profile Counter-counts (≥20%), dominant → builds a profile from a sample.

## 🛠️ Autofix / Annotate
**Autofix**: fix at the **run** level (preserve formatting), preserve case (_match_case), target font = **most common** among allowed; save a **new file**; CLI re-checks the fixed file. **Annotate**: `add_comment` per error paragraph.

## 🖥️ Interfaces
CLI argparse, exit code **1/0** (CI), checks whole folders. Web Flask: /check /fix /annotate /pdf, temp file **deleted immediately** (finally), 25MB, localhost, **no auth**. .exe: PyInstaller + launch.py + **resources.py (sys._MEIPASS)**.

## 🧪 Tests
pytest + the **make_doc** fixture (creates temp .docx). One test file per module; emphasizes **must-not-flag cases**. CI via GitHub Actions.

## ⚖️ Trade-offs
Dictionary spelling: safe, misses things → offset by AI. Skipping math: reduces noise, may miss → --no-skip-math. Only .docx (not .doc).

## 🚀 Future improvements
Extend the dictionary; check tables/figures/TOC; add profiles; GitHub Release .exe; cache AI.

## 💬 Key terms
- **.docx/OOXML / paragraph / run / style / based_on / docDefaults / effective font**
- **python-docx / qn() namespace / section / line_spacing**
- **regex: lookahead/lookbehind/backreference / false positive**
- **Issue (dataclass) / severity / category / runner (DRY)**
- **mathdetect / aggregate_issues / learn_profile (Counter)**
- **autofix (run level) / annotate (add_comment) / exit code**
- **OpenAI-compatible LLM / anti-hallucination / chunking**
- **PyInstaller / sys._MEIPASS / Flask / pytest fixture**

---

## ✅ Pre-interview checklist
- [ ] 30-second pitch.
- [ ] Explain the 4-tier effective font.
- [ ] The false-positive reduction philosophy + an example.
- [ ] The shared-Issue architecture + adding a new check.
- [ ] Format-preserving autofix (run level).
- [ ] Web app safety + .exe packaging (resources/_MEIPASS).
- [ ] 2 limitations + fixes.
- [ ] 2 questions for the interviewer.

**Breathe. You understand this project. Tell your story with confidence. 🍀**
