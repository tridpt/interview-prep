# 7. Interview Question Bank & Sample Answers

70+ questions, grouped, easy to hard. Each has a suggested answer. Rephrase in your own words.

> ⭐ = very commonly asked.

---

## A. Overview & motivation

**A1. ⭐ Briefly introduce your project.**
> Word Checker is an offline tool reviewing .docx files before submission: checks formatting (font/size/spacing/alignment/margins), mechanical text errors, spelling (dictionary + optional AI), headings/numbering, word/page limits. Core: Python + python-docx, a layered architecture (one module per check, errors are a shared Issue), used via a CLI or a Flask web app, with auto-fix/annotate/learn-from-sample, packageable into an .exe. Has a pytest suite + CI.

**A2. Why build this?**
> People lose marks for formatting/spelling errors that are hard to catch by eye. I wanted a tool that auto-reviews, can fix, and reaches non-technical users (web + .exe).

**A3. Who are the users?**
> Students and office workers ensuring documents meet a school/company standard before submission.

**A4. ⭐ Why only .docx, not .doc?**
> .docx is OOXML (a zip of XML), readable by python-docx. The old .doc is a closed binary needing Word/another library. I require resaving as .docx and give a clear error if a .doc is passed.

---

## B. Document reading & OOXML

**B1. ⭐ How is a .docx stored?**
> As Office Open XML — a ZIP containing XML files describing content/formatting/styles. python-docx opens it and provides a high-level API (paragraphs, runs, styles, sections).

**B2. Paragraph vs Run?**
> A paragraph is a block; a run is a same-formatting chunk inside it. Font/size are at the run level, so a paragraph can have multiple runs with different formatting.

**B3. ⭐ What is the "effective font"? Why resolve it?**
> The font Word actually uses for a run. A run may not declare a font; Word takes it in order: run → character style → paragraph style (via based_on) → docDefaults. Reading only run.font.name returns None en masse and over-flags. I resolve all 4 tiers in _resolve_run.

**B4. Why read raw XML for docDefaults?**
> python-docx's high-level API doesn't expose docDefaults; I use document.styles.element + qn("w:docDefaults/...") with the namespace to read the default font/size (size is in half-points, so divide by 2).

**B5. How is line_spacing handled?**
> If it's a multiple (1.5), python-docx returns a float → usable; if it's an absolute distance (Pt) → return None to skip (don't compare mismatched units).

**B6. What is the paragraph "number"?**
> The index of NON-empty paragraphs — so "Paragraph 5" matches the user's perception (empty paragraphs aren't counted).

---

## C. The checks & false-positive reduction

**C1. ⭐ What do text checks use? Give a regex example.**
> Regex: "  +" (double space), "\s+([,.;:!?)])" (extra space before punctuation), "([,;:])(?=\S)" (missing space after, using a lookahead), a backreference for repeated words. Each check has exceptions to avoid false positives.

**C2. ⭐ How do you avoid flagging "3,14" as a missing space?**
> Check the chars before and after the punctuation: if both are digits (number:number) skip it — covering 3,14 and 10:30.

**C3. ⭐ Repeated words: why not flag "xa xa" (reduplication)?**
> Only flag if the repeated word is in the function-word list (_REPEATABLE_TYPO_WORDS like "các","và","the") — words that almost never legitimately repeat. Reduplication isn't in that list, so it's not flagged.

**C4. Missing capitalization: how do you avoid false positives?**
> Skip when the token before the punctuation is an abbreviation (_ABBREVIATIONS), an ordinal ("1."), or an ellipsis ("..."). E.g. "Mục 1. nội dung" isn't flagged.

**C5. ⭐ Why skip math-formula paragraphs?**
> Formulas use their own fonts/sizes and many unusual symbols/spaces → easy false positives. mathdetect.is_math_heavy detects them via math symbols / equation numbers / a math-like token ratio ≥35%, then excludes them from font/size/punctuation checks. Disable with --no-skip-math.

**C6. Alignment: why skip short paragraphs?**
> Paragraphs <12 words are usually titles/captions/signature lines, deliberately centered/right-aligned — not errors. Only check alignment on long-enough paragraphs.

**C7. What's your general error-reporting philosophy?**
> Better to miss than over-flag (reduce false positives). A tool that over-flags gets disabled. So every check has exceptions, and there's a dedicated test set for "must-not-be-flagged" cases.

---

## D. Spelling & AI

**D1. How does offline spelling work?**
> Read common_typos.txt (wrong=right pairs), join into one regex (?<!\w)(...)(?!\w), longest first; each match makes an error Issue with the correct suggestion. Safe because it only flags known pairs.

**D2. ⭐ How does AI spelling avoid hallucination?**
> Call an LLM for a JSON error list, but only accept an error if the wrong phrase ACTUALLY appears in the paragraph (wrong in text). AI-fabricated phrases are dropped. AI errors are rated warning (suggestions).

**D3. Why chunk when calling the AI?**
> To keep the prompt short (~3500 chars/chunk), avoid exceeding context and reduce cost. temperature=0 for stability.

**D4. Is AI mandatory?**
> No, off by default; only enabled with LLM_API_KEY. Without a key the rest runs offline. llm_spelling is imported lazily so non-AI users don't need httpx.

**D5. How do you extend the spelling dictionary?**
> Add a "wrong = right" line to dictionaries/common_typos.txt — no code change.

---

## E. Profiles & learn-from-sample

**E1. What's a profile? How many are built in?**
> A standard dict (font/size/spacing/alignment/margins/tolerances). hanh_chinh (default), luan_van, cong_ty; chosen with --profile.

**E2. ⭐ How does --template learn a standard?**
> learn_profile Counter-counts font/size/spacing/alignment/margins in the sample's body; takes values appearing ≥20% (allowing multiple) plus the most common → builds a profile like config, then checks other files against the sample.

**E3. Why have tolerances?**
> Real measurements are rarely round (spacing 1.49, margin 2.98cm). Tolerances avoid flagging negligible deviations.

---

## F. Auto-fix & annotate

**F1. ⭐ What does --fix fix? Is it safe?**
> Fixes double spaces, extra/missing space around punctuation, trailing space, extra empty paragraphs, dictionary spelling. Safe: never overwrites the original (saves *_fixed.docx), fixes at the run level to preserve formatting, only fixes sure errors.

**F2. ⭐ How do you fix text without breaking formatting?**
> Fix text within each run (a run = same-formatting chunk), so font/color/bold-italic are preserved. Spelling preserves case via _match_case.

**F3. How does --fix-format pick the target font?**
> Pick font/size = the most-used ALLOWED value in the body (Counter), not the first list element — avoiding switching the whole doc to a skewed value.

**F4. What does --annotate do?**
> Creates *_commented.docx: each error paragraph gets a comment (document.add_comment), document-level errors grouped into a comment at the start. Opening in Word shows them in the right margin. The original is untouched.

---

## G. Architecture & interfaces

**G1. ⭐ Describe the architecture.**
> Two interfaces (CLI argparse, Flask web) share runner.run_checks. document.py reads .docx → DocInfo; checker modules return a uniform [Issue]; report/pdf export; autofix/annotate act. Every error is the Issue dataclass.

**G2. ⭐ What's needed to add a new check?**
> Write a check_xxx(doc,...) -> [Issue] function, wire it into runner.run_checks (with an enable flag). All export/actions just work because they share Issue. A plugin-like architecture.

**G3. Do the CLI and web duplicate logic?**
> No — both call run_checks. DRY. They differ only in input handling and output.

**G4. Is the web app safe?**
> Uploads are saved to a temp file then deleted immediately (finally), never stored/sent anywhere, limited to 25MB, localhost only. Limitation: no auth → local use only, don't expose to the Internet.

**G5. ⭐ How do you package the .exe? What's resources.py for?**
> PyInstaller bundles the app + web/ + dictionaries/ into WordChecker.exe. When run from the .exe, data lives in the temp dir sys._MEIPASS; resource_path() checks sys.frozen to return the right path for both dev and .exe. launch.py auto-picks a port + opens the browser.

**G6. Why reconfigure stdout to UTF-8 in the CLI?**
> Windows consoles default to cp1252 and error on Vietnamese; forcing UTF-8 (errors="replace") displays correctly.

---

## H. Testing

**H1. ⭐ How do you test?**
> pytest with a make_doc fixture creating temp .docx files with given formatting (font/size/alignment/spacing) → isolated tests. One test file per checker module.

**H2. ⭐ What's special about the tests?**
> They emphasize false-positive avoidance: tests that "3,14"/"10:30" aren't flagged for missing space, "xa xa" isn't flagged as a repeated word, "Mục 1." isn't flagged for capitalization, consistent quotes aren't flagged as mixed. Both positive and must-not-flag cases.

**H3. What does the make_doc fixture do?**
> Returns a build(paragraphs) function creating a temp .docx in tmp_path via add_para (setting font/size/align/spacing) → no real file needed, fast and isolated.

---

## I. Open-ended / thinking

**I1. ⭐ Hardest challenge?**
> (Hint) Resolving the effective font across 4 OOXML tiers (raw docDefaults XML, walking the style inheritance chain); or designing a host of exceptions to reduce false positives without missing real errors. Problem → approach → result (with tests proving it).

**I2. ⭐ What would you improve with more time?**
> Extend the spelling dictionary; check tables/figures/table-of-contents; add more school profiles; upload a prebuilt .exe to GitHub Releases; add deeper grammar checks; cache AI results.

**I3. Known limitations?**
> Only .docx (not .doc). The dictionary spelling is narrow (safe but misses things). AI sends text to an external service (only enable if accepted). The web has no auth (local only). Page estimation is approximate (350 words/page).

**I4. The technical decision you're most proud of?**
> Resolving the effective font per Word's mechanism + a false-positive reduction philosophy with a dedicated test set — showing I build an actually-usable tool that deeply understands document formatting.

**I5. The biggest trade-off?**
> Dictionary spelling: safe (few false positives) but misses out-of-dictionary errors → offset by the AI option. Skipping math: reduces noise but may miss real errors there → toggleable with --no-skip-math.

**I6. What about very large/many files?**
> The CLI checks whole folders into one report. The web limits to 25MB. python-docx reads into RAM, so huge files may be slow; could optimize with streaming if needed.

---

## J. "Specific code" questions

**J1. What is aggregate_issues for?**
> When the same error repeats in >8 paragraphs, group it into one "— N paragraphs (e.g. paragraph ...)" line for a concise report. DISPLAY only; JSON and annotate keep the full list.

**J2. Why sort spelling phrases longest-first in the regex?**
> To prefer matching longer multi-word phrases before shorter ones (avoid matching part of a longer phrase).

**J3. How does _check_numbering_sequence avoid cascade warnings?**
> After flagging a wrong item (e.g. a 4 where 3 was expected), it updates last_child by the ACTUAL number (4), so the next item (5) isn't cascade-flagged.

**J4. How does autofix delete empty paragraphs?**
> XML op: take the paragraph element (p._element), call getparent().remove(el). Only deletes the excess over max_consecutive.

**J5. Why import llm_spelling inside the runner instead of at the top?**
> Lazy import: only import when do_llm_spelling=True, so non-AI users don't need httpx/config, and startup is faster.

**J6. What determines the CLI exit code?**
> 1 if any Issue has severity == error, else 0 — handy for CI/automation scripts.

---

## 🎤 The 5 most common — rehearse first
1. A1 — Introduce the project.
2. B3 — Effective font & why resolve it.
3. C7/C5 — False-positive reduction & skipping formulas.
4. G1/G2 — Architecture & adding a new check.
5. F1/F2 — Format-preserving auto-fix.

## ❓ Questions YOU should ask the interviewer
- How does the team handle document/format processing (Office, PDF), any internal tools?
- What's the false-positive-reduction / quality-evaluation process for automated tools?
- Is there a need to package/distribute to end users (desktop)?
