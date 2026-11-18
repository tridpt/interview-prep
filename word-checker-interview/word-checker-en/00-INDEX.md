# 📚 Interview Prep Pack — Word Checker Project

Welcome. This pack takes you **from zero to confident** about Word Checker — a tool to check spelling & formatting of Word (.docx) files before submission. Each file digs into one aspect. Read in order.

> All docs were written after **directly reading the real source**: `config.py`, `cli.py`, `webapp.py`, `launch.py`, and the entire `checker/` package (document, formatting, textcheck, spelling, mathdetect, headings, limits, stats, learn_profile, llm_spelling, autofix, annotate, resources, report, runner, issues), plus tests and conftest. No guessing.

---

## 🗺️ Learning path (read in order)

| # | File | Content | Read when |
|---|------|---------|-----------|
| 1 | `01-project-overview.md` | What it is, what it solves, tech stack, usage | First |
| 2 | `02-foundational-concepts.md` | Theory: .docx/OOXML, python-docx, effective font, regex, profiles, math detection, LLM, PyInstaller, pytest | When a term is unclear |
| 3 | `03-system-architecture.md` | Architecture, check flow, the Issue data model, the shared runner | After grasping concepts |
| 4 | `04-checker-deep-dive.md` | Each module in `checker/`, function by function | To study the core |
| 5 | `05-interfaces-deep-dive.md` | CLI, web app (Flask), autofix/annotate, .exe packaging | To study the interfaces & outputs |
| 6 | `06-standout-features.md` | Deep analysis of the "point-scoring" features | To tell an impressive story |
| 7 | `07-interview-questions.md` | 70+ Q&A | To rehearse answers |
| 8 | `08-cheat-sheet.md` | One-page summary for go-time | Last minute |

---

## ⏱️ If you are short on time

- **1 day left**: files 1 → 2 → 3 → 7 → 8.
- **2 hours left**: files 1 → 3 → 8, then skim 7.
- **30 minutes left**: file 8 + the "most common questions" in file 7.

---

## 📦 The project in 3 sentences (memorize)

> Word Checker is an **offline** tool that reviews `.docx` files before submission — checking **formatting** (font, size, line spacing, alignment, page margins), **mechanical text errors** (double spaces, missing/extra spaces, repeated words, sentence capitalization...), **Vietnamese spelling** (a common-typos dictionary + an optional AI/LLM), **headings & numbering**, and **word/page limits**. The core is **Python + python-docx** with a cleanly layered `checker/` package (one module per check type, every issue is a shared `Issue`); usable via a **CLI** or a **drag-and-drop web app (Flask)**, with **auto-fix (--fix)**, **inline comments (--annotate)**, **HTML/PDF/JSON export**, **learning a standard from a sample file (--template)**, and **packaging into an .exe** (PyInstaller) that runs without Python. It has a **full pytest suite** and GitHub Actions CI.

Good luck with your interview! 🍀
