# 5. Interfaces & outputs in detail (CLI, Web app, .exe)

> Both interfaces share the `runner.run_checks` core. Read alongside `cli.py`, `webapp.py`, `launch.py`.

---

## 5.1. `cli.py` — The command-line interface (argparse)

**At startup:** force `stdout/stderr` to UTF-8 (avoiding Vietnamese font errors on Windows cp1252 consoles).

**Functions:**
- `_gather_docx(path)`: if it's a folder → list all `.docx` (skipping temp `~$` files); if a file → return one element.
- `_check_one(path, profile, args)`: call `run_checks` with the `--no-*` flags inverted.
- `_limits_from_args`: take limits from the CLI, falling back to `config.LIMITS`.
- `_suffix_path(src, suffix)`: build the output name (e.g. `bai_fixed.docx`).

**The `run(args)` function — main flow:**
1. Check the file exists; get the `profile` (per `--profile`, or **learn from `--template`** via `learn_profile`, printing the learned profile description).
2. If `--llm-spelling` without a configured key → error 2.
3. Gather the list of valid `.docx` (error if a `.doc` is passed).
4. **Fix mode** (`--fix`/`--fix-format`): only for **one file** (safer/clearer), call `autofix` → print stats → **re-check the fixed file** to report what remains.
5. Run checks + `compute_stats` per file; `print_report` (one file) or per-file + a summary (many files).
6. Export per flags: `--html` (single/`_multi`), `--pdf` (first file), `--json` (`to_dict`), `--annotate` (each file → `*_commented.docx`).
7. **Exit code**: return `1` if any `Issue` has severity error, else `0`.

**`main()`:** defines all argparse flags (`--profile/--template/--fix/--fix-format/--max-words/--min-words/--max-pages/--out/--html/--pdf/--json/--annotate/--no-format/--no-text/--no-headings/--no-skip-math/--no-spelling/--llm-spelling`).

> 💡 A nice touch: `--fix` then **re-checks the fixed file** to show the user what's left; exit code 1/0 for CI.

---

## 5.2. `webapp.py` — The drag-and-drop web app (Flask)

**Setup:** `Flask` with `template_folder`/`static_folder` via `resource_path(...)` (to work even when packaged as an .exe). Upload limit **25MB** (`MAX_CONTENT_LENGTH`). Only accepts `.docx`.

**Helpers:**
- `_save_upload(file_storage)`: save the upload to a **temp file** (`tempfile.mkstemp`).
- `_resolve_profile(req)`: if a `template` file is uploaded → `learn_profile`; else by the profile name in the form.
- `_valid_upload(req)`: check a file exists + `.docx` extension.

**Routes:**
- `GET /` → render `index.html` (passing the profile list, default profile, `llm_enabled`).
- `POST /check` → save temp → `run_checks` (per form flags) + `compute_stats` → return JSON `{filename, summary, stats, issues}` (issues **aggregated** for display) → **delete the temp file** in `finally`.
- `POST /fix` → `autofix` → `send_file(..._fixed.docx)`.
- `POST /annotate` → `run_checks` + `annotate` → `send_file(..._commented.docx)`.
- `POST /pdf` → `run_checks` + `write_pdf` → `send_file(..._bao_cao.pdf)`.
- `GET /favicon.ico` → return the icon.

**Security/privacy:** every route saves the file to a **temp** then **deletes it immediately** (`finally: os.remove`); runs `host="127.0.0.1"` (localhost only); **no authentication** → don't expose to the Internet.

> 💡 A good question: "Is uploading safe?" → The file is only processed in a temp file then deleted, never stored or sent anywhere, limited to 25MB, localhost only. Limitation: no auth → local use only.

---

## 5.3. `launch.py` + `.exe` packaging

- `launch.py`: the launcher for the packaged build — `_free_port()` picks a free port (preferring 5000), opens the browser after 1.2s (a thread), then `app.run`.
- `build_exe.bat` + `WordChecker.spec`: PyInstaller packages the app + **bundles `web/` and `dictionaries/`** into `dist/WordChecker.exe`. Double-clicking runs a local server + opens the browser; copying just the `.exe` to another Windows machine works.
- `resources.py` ensures correct resource paths when running from the `.exe` (`sys._MEIPASS`).

---

## 5.4. `web/index.html` — The UI

A drag-and-drop page: select/drag one or more `.docx`, choose a standard (or upload a sample), click **Check** (calls `/check`, shows the report inline), or **Auto-fix & download** (`/fix`), **Download annotated file** (`/annotate`), **Download PDF** (`/pdf`). Has a **light/dark** toggle.

---

## 5.5. Outputs (summary — details in file 04)

- **Console**: `report.print_report` — summary + aggregated issue list, `[X]` (error) / `[!]` (warning) markers.
- **HTML**: `report.write_html(_multi)` — color-coded table, multi-file version.
- **PDF**: `pdf_report.write_pdf` — fpdf2 + a Unicode Vietnamese font.
- **JSON**: a list of `{file, issues:[issue.to_dict()]}` for automation.

---

## 5.6. Summary of interface technical decisions

| Decision | Reason |
|----------|--------|
| CLI + web share `run_checks` | No duplicated logic (DRY) |
| `--fix` then re-check the fixed file | Show the user what remains |
| Exit code 1/0 | CI/automation integration |
| Web saves temp file then deletes immediately | Privacy; no stored user data |
| `resource_path` for dev + .exe | Same code runs in both environments |
| .exe bundles web+dict | Non-Python users can run it |
| UTF-8 reconfigure stdout | Correct Vietnamese on Windows |
