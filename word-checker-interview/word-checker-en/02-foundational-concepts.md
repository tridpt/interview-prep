# 2. Foundational Concepts (theory)

This file explains **every term** in the project, for someone starting from scratch.

---

## 2.1. The .docx file & OOXML

**What is `.docx`?** The **Office Open XML (OOXML)** format — essentially a **ZIP** file containing several XML files describing content, formatting, styles... (rename `.docx` to `.zip` and extract to see). Different from the old `.doc` (a closed binary format).

**Word structure concepts to know:**
- **Paragraph**: a block of text. Each has a **style**, alignment, line spacing, indent...
- **Run**: a "chunk" of text with **the same formatting** inside a paragraph. A paragraph can have many runs (e.g. a sentence with a few bold words → the bold word is its own run). Font/size are set at the run level.
- **Style**: a named formatting set (Normal, Heading 1, Title...). Styles can **inherit** (based_on) other styles.
- **Section**: a document part holding page setup (margins, paper size).
- **docDefaults**: the document-wide default formatting (default font/size).

---

## 2.2. python-docx

A Python library to read/write `.docx`. In the project:
- `Document(path)` opens a file; `document.paragraphs`, `p.runs`, `p.style`, `p.alignment`, `p.paragraph_format` (line_spacing, first_line_indent, space_before/after), `document.sections[0]` (margins).
- Units: python-docx returns `Length` (Pt, Cm...) — code uses `.pt`, `.cm` to convert.
- Writing: set `run.font.name`, `run.font.size = Pt(...)`, `p.alignment`, `section.top_margin = Cm(...)` then `document.save(out)`.
- Raw XML access via `document.styles.element` + `qn("w:...")` (namespace) to read `docDefaults`, which the high-level API doesn't expose.

---

## 2.3. "Effective font" — the hardest problem

**The problem:** a run **may not declare a font/size directly**. Word then takes the font in priority order:
1. Font set directly on the **run**.
2. If absent → from the run's **character style**.
3. If still absent → from the **paragraph style** (following the `based_on` chain).
4. Finally → **docDefaults** (document default).

**The solution in `document.py`:** `_resolve_run` tries all 4 sources in order; `_style_font` walks the style inheritance chain (`based_on`) for the first value found; `_doc_default_font` reads the `docDefaults` XML. → Each run gets a precise **effective** font/size, just as Word actually displays.

> 💡 This is the most "point-scoring" technical detail to mention: "I don't just read the font directly on the run but resolve the real *effective* font across the 4 OOXML inheritance tiers — otherwise it would produce mass false positives."

---

## 2.4. Regex — the main tool for text checks

`textcheck.py` and `spelling.py` use lots of **regex** (`import re`) to find error patterns:
- `r"  +"` → multiple consecutive spaces (double space).
- `r"\s+([,.;:!?)])"` → extra space before punctuation.
- `r"([,;:])(?=\S)"` → missing space after punctuation (using a **lookahead** `(?=...)` so it doesn't "consume" the next char).
- `r"(?<![^\W\d_])([^\W\d_]+)(\s+)(\1)(?![^\W\d_])"` → adjacent repeated words (using a **backreference** `\1` and **lookbehind/lookahead** to bound at word edges).
- Spelling: join all wrong words into one pattern `(?<!\w)(a|b|c)(?!\w)`, **longest first** to prefer multi-word phrases.

**Regex concepts to know:** lookahead `(?=)`, lookbehind `(?<=)`, negative `(?!)`/`(?<!)`, backreference `\1`, flags `IGNORECASE`/`UNICODE`. This is a part you may be quizzed on deeply.

---

## 2.5. Standard profiles & "learn from a sample"

- **Profile**: a dict describing the standard (allowed fonts, sizes, line spacing, alignment, margins, tolerances...). `config.PROFILES` has `hanh_chinh`, `luan_van`, `cong_ty`.
- **Tolerance**: because real measurements are rarely round (line spacing 1.49 vs 1.5, margin 2.98cm vs 3.0cm), each check has a **tolerance** to avoid false positives.
- **Learning from a sample** (`learn_profile.py`): given a correctly-formatted `.docx`, the tool **counts** the most common font/size/spacing/alignment in the body (using `Counter`, an occurrence threshold of ≥20%) and builds a profile → then checks other files against that sample. The "no manual config editing" approach.

---

## 2.6. Math formula detection (noise reduction)

**The problem:** academic documents with many formulas (using their own fonts/sizes, many unusual symbols, many spaces around symbols) easily produce **false positives** on font/punctuation checks.

**The solution in `mathdetect.py`:** `is_math_heavy(text)` treats a paragraph as a "formula" if: (1) it contains a strong math symbol (∑∫√≤≥α…), or (2) it has an equation number at line end `(4.1)`, or (3) the **ratio of math-like tokens** (single-letter variables, functions `f(x)`, subscripts `x_i`, operators `=+-*/^`) is ≥35%. Font/size/punctuation checks **skip** these paragraphs (toggle with `--no-skip-math`).

> 💡 A great example of a **false-positive reduction** mindset — a tool that over-flags will be disabled by users.

---

## 2.7. The Issue dataclass & severity/category

- Every error is a unified **`@dataclass Issue`**: `category`, `severity`, `message`, `paragraph`, `excerpt`, `suggestion`.
- **severity**: `error` (almost certainly wrong, should fix) or `warning` (worth reviewing).
- **category**: Formatting / Text / Spelling / Spelling (AI) / Headings / Limits.
- One shared structure lets every module "speak the same language", and the export parts (HTML/PDF/JSON/annotate) handle them uniformly.

---

## 2.8. LLM for spelling (optional) & anti-hallucination

- `llm_spelling.py` calls an LLM in the **OpenAI-compatible** standard (Groq/Gemini/OpenAI) via `httpx`, requesting a **JSON** list of errors.
- **Chunking** by character count so the prompt isn't too long; `temperature=0` for stability.
- **Anti-hallucination**: only accept an error if the **wrong phrase actually appears** in the paragraph (`if wrong not in text_by_num[num]: continue`) → AI-fabricated errors are dropped. AI errors are rated **warning** (suggestions only).
- Off by default; only enabled when `LLM_API_KEY` is set.

---

## 2.9. PyInstaller & .exe packaging

- **PyInstaller** packages a Python app + dependencies into an `.exe` that runs without installing Python.
- When run from the `.exe`, data files (`web/`, `dictionaries/`) are extracted to a temp dir `sys._MEIPASS`. So `resources.py` → `resource_path()` resolves paths correctly **for both dev and packaged** (checking `sys.frozen`).
- `launch.py` is the launcher for the `.exe`: auto-pick a free port, open the browser, run Flask.

---

## 2.10. pytest & fixtures

- Tests use **pytest** with a **fixture** `make_doc` (in `conftest.py`): it returns a `build(paragraphs)` function that creates a temp `.docx` with given formatting (font/size/alignment/spacing) in `tmp_path` → **isolated** tests with no real file needed.
- Each checker module has a corresponding test file (`test_textcheck.py`, `test_formatting.py`, ...), testing both the **positive** case (catches the error) and the **no-false-positive** case (e.g. "3,14" not flagged for a missing space, the reduplication "xa xa" not flagged as a repeated word).

---

## 2.11. Other terms

- **Run**: a same-formatting chunk of text (see 2.1).
- **False positive**: flagging an error where it's actually correct — the project heavily prioritizes reducing this.
- **Aggregate**: when the same error repeats across many paragraphs (>8), the report groups them into one "— N paragraphs" line for brevity (display only; JSON/annotate keep the full list).
- **Exit code**: the program's exit status; 1 if there are serious errors, 0 if clean — handy for automation/CI.
- **OpenAI-compatible**: OpenAI's standard API format that Groq/Gemini also follow → swapping providers only needs base_url/key/model.
