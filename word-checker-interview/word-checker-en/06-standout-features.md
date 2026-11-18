# 6. Deep Analysis of the "Point-Scoring" Features

Each feature: the idea, how it's done, "why it's cool".

---

## 6.1. Resolving the "effective font" in OOXML ⭐⭐ (the most distinctive)

**Idea:** report the exact font/size Word *actually displays*, not just the one declared directly on the run.

**How:** `_resolve_run` tries 4 tiers per OOXML's mechanism: run → character style → paragraph style (following `based_on`) → docDefaults (read from raw XML). `_style_font` walks the inheritance chain with loop protection.

**Why it's cool:** this is the **hardest problem** of the project. Reading only `run.font.name` returns None for most runs (the font lives in the style) → either mass false positives or missed errors. Deep understanding of Word's structure is very impressive.

**Possible question:** *"Why read raw XML for docDefaults?"* → because python-docx's high-level API doesn't expose docDefaults; you must use `qn()` + the namespace.

---

## 6.2. The false-positive reduction philosophy ⭐⭐

**Idea:** a tool that over-flags gets disabled. So **better to miss than over-flag**.

**How — many deliberate exceptions:**
- Skip **math-formula paragraphs** (`mathdetect`) for font/size/punctuation checks.
- "Missing space after punctuation" **skips number:number** ("3,14", "10:30").
- "Repeated words" **only flags function words** ("các các"), not **reduplication** ("xa xa").
- "Missing capitalization" **blocks** abbreviations, ordinals, ellipses.
- Alignment **skips short paragraphs <12 words** (titles/captions are deliberately centered).
- Dictionary spelling **only flags known pairs**.
- AI spelling has **anti-hallucination** (the phrase must exist in the paragraph).
- A **dedicated test set** for "must-not-be-flagged" cases.

**Why it's cool:** shows real product thinking, not just "it works". A very impressive story to tell.

---

## 6.3. Learning a standard from a sample (--template) ⭐

**Idea:** don't make users edit config — give it a correctly-formatted `.docx` and the tool derives the standard.

**How:** `learn_profile` `Counter`-counts font/size/spacing/alignment/margins/indent in the body; takes values appearing ≥20% (allowing multiple valid values) plus the most common one → builds a profile just like config.

**Why it's cool:** turns a manual config problem into "just give a sample" — great UX, and demonstrates simple-but-effective statistics.

---

## 6.4. Auto-fix that preserves formatting (--fix/--fix-format) ⭐

**Idea:** not just report errors but fix them, **without breaking** existing formatting.

**How:** fix at the **run** level (preserving each chunk's font/color/bold-italic); spelling **preserves case** (`_match_case`); when fixing formatting, pick the **target font/size = the most-used allowed value** instead of the first list element (avoid switching the whole doc to a skewed value). Always **save a new file**, never touching the original; the CLI **re-checks the fixed file**.

**Why it's cool:** fixing text without breaking formatting is subtle; "pick the most-common target font" shows thinking about real consequences.

---

## 6.5. An extensible architecture + one core for two interfaces ⭐

**Idea:** adding a new check is easy; the CLI and web don't duplicate code.

**How:** every error is a shared `Issue`; each check is a `check_xxx(doc,...) -> [Issue]`; `runner.run_checks` bundles them; the CLI/web both call the runner. Adding a check = a function + wiring it in, and all export/actions just work.

**Why it's cool:** demonstrates good software design (SOLID-ish, separation of concerns, DRY).

---

## 6.6. AI spelling with anti-hallucination ⭐

**Idea:** use an LLM to catch errors beyond the dictionary, without letting the AI fabricate.

**How:** chunk, prompt for JSON, `temperature=0`; **only accept an error if the wrong phrase actually appears** in the paragraph; rate as warning; off by default.

**Why it's cool:** **controlled** LLM integration — the golden rule with AI is not to trust output blindly.

---

## 6.7. Exporting a .docx with inline comments at errors (--annotate)

**Idea:** return errors right inside the Word file so users see comments in the right margin when they open it in Word.

**How:** `document.add_comment(p.runs, ...)` per error paragraph; document-level errors grouped into a comment at the start; the original is untouched. Requires re-creating the paragraph numbering to map correctly.

**Why it's cool:** a very friendly experience — fix right in the familiar environment (Word).

---

## 6.8. .exe packaging for non-Python users

**Idea:** spread the tool to non-technical users.

**How:** PyInstaller bundles the app + `web/` + `dictionaries/`; `launch.py` opens the browser; `resources.py` handles `sys._MEIPASS` paths. Double-click the `.exe` to run.

**Why it's cool:** thinking about **distribution** — many projects forget this step.

---

## 6.9. "Feature ↔ technical concept" table (memorize)

| Feature | Concept you demonstrate |
|---------|-------------------------|
| Effective font | OOXML, style inheritance, raw XML reading, the qn() namespace |
| False-positive reduction | advanced regex, exception design, anti-false-positive tests |
| Learn from a sample | Counter statistics, threshold, config generation |
| Format-preserving autofix | run-level ops, case preservation, most-common target value |
| Extensible architecture | a shared dataclass, separation of concerns, DRY (shared runner) |
| AI spelling | OpenAI-compatible LLM, chunking, anti-hallucination |
| Annotate | python-docx add_comment, paragraph mapping |
| .exe packaging | PyInstaller, sys._MEIPASS, distribution |
| math detection | symbol/token-ratio heuristics |
| stats/limits | page/reading-time estimation |

> 💡 **Storytelling strategy:** if asked "most proud of?", pick the **4-tier effective font** (document-processing depth) or the **false-positive reduction philosophy** (product thinking). Both show you build a tool that's *actually usable*, not just a demo.
