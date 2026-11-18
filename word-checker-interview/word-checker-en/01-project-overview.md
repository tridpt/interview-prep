# 1. Project Overview — Word Checker

## 1.1. What is Word Checker?

Word Checker is a **tool that reviews Word (.docx) files** before submission — for students and office workers who need their assignments/reports to follow formatting standards and be error-free. You feed it a `.docx` file, and it returns a list of issues: wrong font/size, incorrect line spacing/alignment/page margins, double spaces, missing space after a comma, repeated words, missing sentence capitalization, spelling errors, heading level jumps, exceeding the word limit... plus statistics (word count, estimated pages, reading time).

### Core points to remember
- **Runs offline** (except the optional AI feature). Data is sent nowhere.
- Core: Python + **python-docx** to read the Word structure.
- **Cleanly layered architecture**: one module per check type, every issue is a shared `Issue` dataclass.
- Two interfaces: a **CLI** and a **drag-and-drop web app (Flask)**.
- Not only detects but also **fixes** (`--fix`/`--fix-format`), **annotates the file** (`--annotate`), and **learns a standard from a sample** (`--template`).
- Can be packaged into an **`.exe`** (PyInstaller) so non-Python users can run it.

## 1.2. What problem does it solve?

1. **Losing marks for formatting/spelling errors** that are hard to catch by eye → fully automated review.
2. **Every school/company has a different standard** → multiple built-in **profiles** + **learning from a sample file**.
3. **Manual fixing is slow** → auto-fix mechanical/spelling/formatting errors, saving to a new file (never touches the original).
4. **Non-technical users** → drag-and-drop web app + a runnable `.exe`.
5. **Demonstrating technical ability**: parsing the complex OOXML structure ("effective font"), designing an extensible check system, smart noise reduction (skipping math formulas), integrating an LLM with anti-hallucination, and distribution packaging — the point-scorer.

> 💡 **Interview tip:** this is a **document-processing + good software architecture** project. Emphasize: (1) solving the hard "effective font" problem in OOXML; (2) an architecture that makes adding a new check trivial; (3) a **false-positive reduction** mindset — skipping math, only flagging sure errors, preventing AI hallucination.

## 1.3. Check groups (what it checks)

| Group | Module | Example errors caught |
|-------|--------|-----------------------|
| **Formatting** | `formatting.py` | wrong font/size, alignment, line spacing, page margins, first-line indent, inconsistent fonts/sizes |
| **Text (mechanical)** | `textcheck.py` | double spaces, extra/missing space around punctuation, trailing/in-paren spaces, extra empty paragraphs, repeated words, missing capitalization, `--` instead of an en-dash, mixed straight/curly quotes, placeholders (TODO, Lorem ipsum...) |
| **Spelling** | `spelling.py` | matches against `common_typos.txt` (safe, few false positives) |
| **AI spelling** | `llm_spelling.py` | uses an LLM for broader detection (optional, anti-hallucination) |
| **Headings** | `headings.py` | heading level jumps, non-sequential numbering |
| **Limits** | `limits.py` | over/under word count, exceeding estimated pages |

## 1.4. Tech stack

**Core (Python 3.10+):**
- **python-docx** `>=1.1` — read/write `.docx`, parse paragraphs/runs/styles/sections.
- **Flask** `>=3.0` — the drag-and-drop web app.
- **fpdf2** `>=2.7` — PDF report export (Unicode Vietnamese font).
- **httpx** + **python-dotenv** — the optional AI spelling feature (OpenAI-compatible).
- Standard library: `re` (regex — the heart of textcheck/spelling), `argparse` (CLI), `csv`, `json`, `dataclasses`, `collections.Counter`.

**Packaging & testing:**
- **PyInstaller** — package into `WordChecker.exe` (bundles `web/` + `dictionaries/`).
- **pytest** — the test suite; **Pillow** (dev) to generate the icon/logo.
- **GitHub Actions** — CI running tests.

> 💡 **A trap question: "Why only .docx, not .doc?"** Answer: ".docx is the open Office Open XML (OOXML) format — essentially a zip of XML files that python-docx can read. The old .doc is a closed binary format needing Word/another library. I require resaving as .docx, and the CLI gives a clear error if you pass a .doc."

## 1.5. Usage (representative commands)

```bash
pip install -r requirements.txt

# Basic check
python cli.py assignment.docx

# Export HTML/PDF/JSON reports
python cli.py assignment.docx --html report.html --pdf report.pdf --json result.json

# Pick a profile, or learn the standard from a sample
python cli.py assignment.docx --profile luan_van
python cli.py assignment.docx --template standard_sample.docx

# Auto-fix (never overwrites the original), also fix formatting
python cli.py assignment.docx --fix --fix-format

# Annotate the file, check limits, enable AI
python cli.py assignment.docx --annotate
python cli.py assignment.docx --max-words 2000 --min-words 500
python cli.py assignment.docx --llm-spelling

# Check a whole folder, combine into one report
python cli.py folder --html combined_report.html

# Drag-and-drop web app
python webapp.py     # opens http://127.0.0.1:5000
```
The tool **exits with code 1** if there are serious errors (handy for CI/automation), `0` if the file is clean.

## 1.6. Project structure

```
word-checker/
├─ cli.py            # CLI entry point (argparse)
├─ webapp.py         # drag-and-drop web app (Flask)
├─ launch.py         # launcher for the .exe (auto-opens the browser)
├─ build_exe.bat / WordChecker.spec  # PyInstaller packaging
├─ config.py         # formatting standards (profiles), enable/disable each check
├─ web/index.html    # drag-and-drop UI
├─ dictionaries/common_typos.txt   # spelling-typo dictionary
├─ tests/            # pytest suite
└─ checker/          # THE CORE: one module per check type
   ├─ document.py    # read .docx, resolve "effective font"
   ├─ issues.py      # the Issue dataclass (one error)
   ├─ runner.py      # bundles the check logic (shared by CLI + web)
   ├─ formatting.py / textcheck.py / spelling.py / llm_spelling.py
   ├─ headings.py / limits.py / stats.py / mathdetect.py
   ├─ learn_profile.py   # learn a standard from a sample file
   ├─ autofix.py / annotate.py    # fix & annotate
   ├─ report.py / pdf_report.py   # HTML/PDF export
   └─ resources.py   # resolve resource paths (dev + .exe)
```

> ⚠️ **Security note (worth raising proactively):** the web app **runs locally with no authentication**; uploaded files are processed in a temp file then deleted, never stored, never sent anywhere. The README clearly says not to expose this port to the public Internet. The AI feature sends text to an LLM service, so only enable it if you accept that.
