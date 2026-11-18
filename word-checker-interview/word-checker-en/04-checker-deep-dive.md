# 4. The `checker/` core in detail (module by module, function by function)

> Written after directly reading the entire `checker/` package. Read alongside the real code.

---

## 4.1. `issues.py` — The error structure

Defines `SEVERITY_ERROR/WARNING`, the `CATEGORY_*` constants, and:
```python
@dataclass
class Issue:
    category: str; severity: str; message: str
    paragraph: int | None = None; excerpt: str = ""; suggestion: str = ""
    def location(self): return "whole document" if paragraph is None else f"Paragraph {paragraph}"
    def to_dict(self): ...   # for JSON
```
**The heart of the architecture**: every module speaks the same "language" via `Issue`.

---

## 4.2. `document.py` — Reading .docx & resolving the effective font (the hardest)

- `RunInfo(text, font_name, font_size)`, `ParagraphInfo(...)`, `DocInfo(...)` — dataclasses describing the "digested" document.
- **`_doc_default_font(document)`**: reads raw XML `w:docDefaults/w:rPrDefault/w:rPr` via `qn(...)` for the default font/size (the high-level API doesn't expose it). Size is in half-points, so `/2`.
- **`_style_font(style)`**: walks the style inheritance chain (`base_style`) for the first font/size, with loop protection (`seen`).
- **`_resolve_run(run, paragraph, defaults)`** ⭐: resolves the **effective** font/size across 4 tiers: run → character style → paragraph style → docDefaults.
- **`_line_spacing(p)`**: returns line spacing if it's a **multiple** (a float like 1.5); if it's an absolute distance (Pt) → None (skip).
- **`_para_indent_spacing(p)`**: first-line indent (cm), space before/after (pt).
- **`load_document(path)`** ⭐: iterates `document.paragraphs`, builds a `ParagraphInfo` per paragraph (counting `number` = the index of **non-empty** paragraphs, the number the user sees), gathers the effective `fonts`/`sizes`; reads margins from `sections[0]`. Wraps the XML/margin reads in `try/except` so it doesn't crash on odd files.

> 💡 Saying "the 4-tier effective font" + "counting paragraphs excluding empties to match the user's perception" is a big plus.

---

## 4.3. `mathdetect.py` — Formula detection (noise reduction)

`is_math_heavy(text)` returns True if: (1) it contains a symbol in `_MATH_SYMBOLS` (∑∫√≤≥, Greek letters...); (2) it matches `_EQ_LABEL` (an end-of-line equation number like `(4.1)`); (3) the **ratio of math-like tokens ≥ 0.35** (single-letter variables, functions `f(x)`, subscripts `x_i`, pure operators `=+-*/^()`). Used to exclude formula paragraphs from font/size/punctuation checks.

---

## 4.4. `formatting.py` — Formatting checks

`check_formatting(doc, profile, skip_math)`:
- Take `body` = non-empty, non-heading paragraphs; further filter `prose` = drop `is_math_heavy` paragraphs (if skip_math).
- Each prose paragraph runs: `_check_font` (font not in the allowed list → **error**), `_check_size` (size not allowed → error), `_check_alignment` (alignment; **skips short paragraphs <12 words** since titles/captions are often deliberately centered → warning), `_check_line_spacing` (with tolerance), `_check_indent_spacing` (indent/space before-after if the profile enables it).
- `_check_consistency(prose)`: uses `Counter` to count fonts/sizes; if the body uses >1 font or >1 size → **one** "inconsistent" warning.
- `_check_margins(doc, profile)`: compares page margins with tolerance.

> A subtle detail: alignment is only flagged for paragraphs ≥12 words → avoids false-positives on titles/signature lines.

---

## 4.5. `textcheck.py` — Mechanical text errors (regex)

`check_text(doc, checks, max_consecutive_empty, skip_math)` iterates each paragraph (skipping empty & formula paragraphs), running the sub-functions enabled in `config.TEXT_CHECKS`:
- `_check_double_space` (`  +`), `_check_space_before_punct` (`\s+([,.;:!?)])`), `_check_missing_space_after_punct` (`([,;:])(?=\S)` — **skips number:number** like "3,14"/"10:30"), `_check_trailing_space`, `_check_space_in_parens`.
- `_check_repeated_words`: regex backreference; **only flags function words** in `_REPEATABLE_TYPO_WORDS` ("các","và","the"...) → avoids flagging **reduplication** ("xa xa").
- `_check_sentence_capitalization`: lowercase after `.!?` → warning; **blocks** abbreviations (`_ABBREVIATIONS`), ordinals ("1."), ellipsis ("...").
- `_check_double_hyphen` (`--`), `_check_placeholder` (Lorem ipsum/TODO/FIXME/`[...]`/`xxxx`).
- Document-level: `_check_mixed_quotes` (mixing straight `"` and curly `" "`), `_check_empty_paragraphs` (count consecutive empties over the threshold).

> 💡 The whole module illustrates the **false-positive reduction** mindset: every check has exceptions so it doesn't flag spuriously.

---

## 4.6. `spelling.py` — Dictionary-based spelling (offline)

- `load_typo_dict()`: reads `dictionaries/common_typos.txt` (each line `wrong = right`, skipping `#` lines).
- `_build_pattern(typos)`: joins all wrong words into one regex `(?<!\w)(...)(?!\w)`, **longest first** (prefer multi-word phrases), flags IGNORECASE|UNICODE.
- `check_spelling(doc)`: scans each paragraph, each match makes an **error** `Issue` with the correct suggestion.
- Philosophy: **safe, few false positives** — only flags known typo pairs.

---

## 4.7. `llm_spelling.py` — AI spelling (optional)

- `check_spelling_llm(doc)`: gather paragraphs → **chunk** (`_chunks`, ~3500 char limit) → prompt requesting JSON `{p, wrong, correct, note}`.
- `_call_llm`: call OpenAI-compatible via httpx (`temperature=0`, `max_tokens=1500`), wrap errors → `LLMError`.
- `_extract_json_array`: extract the JSON array even when the model wraps it in ```` ```json ````.
- **Anti-hallucination** ⭐: only accept an error if `wrong` **actually appears** in the paragraph (`if wrong not in text_by_num[num]: continue`); rated **warning**.

---

## 4.8. `headings.py` — Headings & numbering

- `_check_level_jumps`: tracks heading levels; if `level > prev_level + 1` (e.g. 1→3) → warns about a missing intermediate level.
- `_check_numbering_sequence`: parses the section number in the text (`_NUM_RE`), checks each parent's children are sequential (1,2,4 → missing 3); updates by actual number to **avoid cascade warnings**.

---

## 4.9. `limits.py` & `stats.py`

- `stats.compute_stats(doc)`: counts words/chars/paragraphs/headings; estimates pages (`350 words/page`) and reading time (`200 words/min`).
- `limits.check_limits(doc, limits)`: compares word/page counts against `min_words/max_words/max_pages` → **error** `Issue`s with "cut/add N words" suggestions.

---

## 4.10. `learn_profile.py` — Learn a standard from a sample

- `learn_profile(path)`: `Counter`-counts font/size/spacing/alignment/margins/indent in the sample's body.
- `_common_values(counter, total, threshold=0.2)`: take values appearing ≥20% (always including the most common) → allows multiple valid fonts/sizes.
- `_dominant(counter)`: the most common value (for spacing/alignment/margins).
- Returns a **profile** in the same shape as config → checks other files against the sample. `describe_profile` prints a summary for the user.

---

## 4.11. `autofix.py` — Auto-fix (preserving formatting)

- **Safety principles**: never overwrite the original; fix at the **run** level to preserve font/color/bold-italic; only fix sure errors.
- `_fix_run_text(text, typos)`: remove space before punctuation, collapse double spaces, add space after `,;:`, fix dictionary typos (**preserving case** via `_match_case`).
- `_remove_extra_empty_paragraphs`: delete extra empty paragraphs (XML op `el.getparent().remove(el)`).
- `_fix_formatting(document, profile)` (with `--fix-format`): pick the **target font/size = the most-used allowed value** (avoid setting to the first list element, which would skew), reset alignment/line spacing/indent/space; `_fix_margins` sets page margins.
- `autofix(...)` returns stats of fixes made.

> 💡 "Fix at the run level to not break formatting" and "pick the most-common target font instead of the first list element" are two notable subtleties.

---

## 4.12. `annotate.py` — Inline comments in the .docx

- Re-creates the paragraph numbering (`number`) like `load_document` to map `Issue.paragraph` back to the right paragraph.
- Groups issues by paragraph; each error paragraph → `document.add_comment(p.runs, text=..., author="Word Checker")`. Document-level errors are grouped into one comment at the first paragraph. The original file is untouched.

---

## 4.13. `report.py`, `pdf_report.py`, `resources.py`, `runner.py`

- `report.aggregate_issues(issues, threshold=8)` ⭐: **aggregates repeated** same-message errors when >8 paragraphs into a single "— N paragraphs (e.g. paragraph ...)" line for brevity — **display only**; JSON/annotate keep the full list.
- `report.print_report/write_html/write_html_multi`: console + HTML (severity-colored, multi-file version).
- `pdf_report.write_pdf`: fpdf2 + a Unicode font to print Vietnamese correctly.
- `resources.resource_path`: resolves paths for both dev (`__file__/..`) and the .exe (`sys._MEIPASS`).
- `runner.run_checks`: orchestrates everything per flags (described in file 03); **lazily imports** `llm_spelling` only when needed.

---

## 4.14. The test suite (tests/)

- `conftest.py`: the `make_doc` fixture creates a temp `.docx` with given formatting → isolated tests.
- One test file per module; tests both the **positive** case and the **no-false-positive** case: `test_missing_space_not_flag_numbers` ("3,14"/"10:30" not flagged), `test_repeated_word_ignores_reduplication` ("xa xa" not flagged), `test_sentence_cap_ignores_abbrev_number` ("Mục 1." not flagged for capitalization), `test_no_mixed_quotes_when_consistent`...

> 💡 The suite **specifically emphasizes false positives** — reflecting the project's exact philosophy.

---

## 4.15. Summary of core technical decisions

| Decision | Reason |
|----------|--------|
| A shared `Issue` for every module | All export/actions handle uniformly, easy to extend |
| 4-tier effective font resolution | Matches Word's display, avoids font false positives |
| `mathdetect` skipping formulas | Reduces false positives on academic docs |
| Every check has exceptions | "Better to miss than over-flag" philosophy |
| AI anti-hallucination (phrase must exist) | Don't accept AI-fabricated errors |
| autofix at run level + most-common target font | Preserves formatting, no skew |
| `aggregate_issues` display-only | Concise report but full JSON/annotate |
| Shared runner + lazy LLM import | DRY + httpx not required when unused |
