# 3. System Architecture

## 3.1. High-level diagram

```
┌─────────────── INTERFACES (2 ways to use, sharing one core) ───────────────┐
│  cli.py (argparse)                 webapp.py (Flask, drag-and-drop)         │
│   --html/--pdf/--json/--fix/...     /check /fix /annotate /pdf              │
│         │                                   │                              │
│         └──────────────┬────────────────────┘                              │
│                        ▼                                                    │
│              checker/runner.py  ── run_checks(path, profile, do_*) ──▶ [Issue]│
│                        │                                                    │
│   ┌────────────────────┼───────────────────────────────────────────┐       │
│   ▼          ▼         ▼          ▼          ▼          ▼            ▼       │
│ document  formatting textcheck spelling   headings   limits   llm_spelling   │
│ (read docx,(formatting)(mechanical)(dict) (headings)(words/  (AI, optional) │
│  effective font)                            pages)                          │
│   │                          ▲                                              │
│   │                          │ mathdetect (skip formulas)                   │
│   ▼                                                                         │
│ DocInfo (paragraphs, margins, default font)                                 │
│                                                                             │
│  Outputs: report.py (HTML/console) · pdf_report.py (PDF) ·                  │
│           autofix.py (fix) · annotate.py (comments) · JSON                  │
│  Helpers: issues.py (Issue) · stats.py · learn_profile.py · resources.py    │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Core trait:** a **shared runner** — both the CLI and the web app call `run_checks()`, so the check logic **isn't duplicated**. Each check type is an independent module returning a uniform list of `Issue`s.

## 3.2. Layers & responsibilities

| Layer | File | Responsibility |
|-------|------|----------------|
| Interfaces | `cli.py`, `webapp.py`, `launch.py` | Take input/args, call the runner, export results |
| Orchestration | `checker/runner.py` | Bundle the check steps per enable/disable flags |
| Document reading | `checker/document.py` | Read .docx → `DocInfo` (effective font, margins, paragraphs) |
| Checks | `formatting`, `textcheck`, `spelling`, `llm_spelling`, `headings`, `limits` | One error group per module, returns `[Issue]` |
| Noise reduction | `mathdetect.py` | Detect formula paragraphs to skip |
| Issue structure | `issues.py` | The shared `Issue` dataclass |
| Output | `report.py`, `pdf_report.py` | HTML/console/PDF |
| Actions | `autofix.py`, `annotate.py` | Fix the file, add comments |
| Support | `stats.py`, `learn_profile.py`, `resources.py` | Stats, learn standard, paths |
| Config | `config.py` | Profiles, check toggles, LLM |

**Point for the interview:** a **plugin-like** architecture — adding a new check just needs a `check_xxx(doc, ...) -> [Issue]` function wired into `runner.py`. Everything downstream (HTML/PDF/JSON export, annotate, aggregation) works automatically because everything shares `Issue`.

## 3.3. Data model

### `DocInfo` (produced by `document.py`)
```python
DocInfo(
  path, paragraphs: list[ParagraphInfo], margins_cm: dict,
  default_font, default_size
)
ParagraphInfo(
  index, number,          # number = index of NON-empty paragraphs (shown to the user)
  text, style_name, is_heading,
  alignment, line_spacing, first_line_indent_cm, space_before_pt, space_after_pt,
  fonts: set, sizes: set, # EFFECTIVE fonts/sizes in the paragraph
  runs: list[RunInfo]
)
```
➡️ `document.py` has "digested" the complex .docx into a tidy structure for the check modules — **separating parsing from check logic**.

### `Issue` (produced by every check module)
```python
Issue(category, severity, message, paragraph, excerpt, suggestion)
# severity: "error" | "warning"
# category: Formatting | Text | Spelling | Spelling (AI) | Headings | Limits
```

## 3.4. Check flow (run_checks)

```
load_document(path) → DocInfo
  ├─ do_format     → check_formatting(doc, profile, skip_math)
  ├─ do_text       → check_text(doc, TEXT_CHECKS, MAX_EMPTY, skip_math)
  ├─ do_headings   → check_headings(doc, HEADING_CHECKS)
  ├─ do_spelling   → check_spelling(doc)         # dictionary
  ├─ do_llm        → check_spelling_llm(doc)      # AI (lazy import, only when on)
  └─ always        → check_limits(doc, limits)
→ combine all into list[Issue]
```
Note: `llm_spelling` is **imported lazily** (only when `do_llm_spelling=True`) so httpx isn't required/touched when unused.

## 3.5. Output action flows

| Action | File | How |
|--------|------|-----|
| Console | `report.print_report` | Summary + (aggregated) issue list, `[X]`/`[!]` markers |
| HTML | `report.write_html(_multi)` | Color-coded issue table by severity; multi-file version |
| PDF | `pdf_report.write_pdf` | fpdf2 + a Unicode Vietnamese font |
| JSON | in `cli.py` | `issue.to_dict()` for automation |
| Fix | `autofix.autofix` | Fix at the **run** level to preserve formatting; save a new file |
| Annotate | `annotate.annotate` | `document.add_comment(...)` per error paragraph |

## 3.6. Two interfaces, one core

- **CLI** (`cli.py`): `argparse` parses flags → `run_checks` → print/export. Supports folder checking, exit code 1/0.
- **Web app** (`webapp.py`): Flask, routes `/check`, `/fix`, `/annotate`, `/pdf`; saves the upload to a **temp file** then **deletes it immediately** after processing (`finally: os.remove`). Also calls `run_checks`.
- **`.exe`** (`launch.py` + PyInstaller): runs Flask + auto-opens the browser; `resources.py` handles resource paths when packaged.

## 3.7. Why is this architecture sound?

- **DRY**: a shared runner → CLI and web don't duplicate logic.
- **Easy to extend**: a new check = a function returning `[Issue]`.
- **Separation of concerns**: document reading ≠ check logic ≠ output ≠ actions.
- **Data safety**: offline, never overwrites the original, web deletes temp files.
- **Easy distribution**: .exe packaging for non-Python users.
