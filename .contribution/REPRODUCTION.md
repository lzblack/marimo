# Reproduction — marimo #5832

- **Issue:** [PDF download in the presence of css with background colour (#5832)](https://github.com/marimo-team/marimo/issues/5832)
- **Type:** `enhancement` + `help wanted` (open) — a feature request, not a crash/bug.
- **Maintainer status:** Myles Scolnick (core) welcomed a contribution and added `help wanted`; Light2Dark (member) reviewed the proposed approach ("sounds reasonable") and suggested validating against `kitchen_sink.py`.

## Reproduction Process

### Environment Setup

Started on native Windows but hit the frontend build wall: `make fe` runs `scripts/buildfrontend.sh`, which native PowerShell can't execute. marimo's CONTRIBUTING also explicitly recommends WSL for Windows. Migrated the whole setup to **WSL2 (Ubuntu)** and re-cloned into the Linux filesystem (`~/repos/marimo`, *not* `/mnt/c`, to avoid slow IO and line-ending/permission issues).

Toolchain (all met marimo's requirements after migration): `uv` 0.11.19, Node v24.16.0 (req ≥22), pnpm 10.28.2 (req ≥10), GNU Make 4.3, gcc 13.3.

Issues hit and how they were resolved:

- **`ModuleNotFoundError: No module named 'msgspec'`.** The editable install was missing a core runtime dep. `uv run` automatically created/synced the project `.venv` (equivalently `uv pip install --group=dev -e .`), which pulled in `msgspec` and the rest.
- **Fork behind upstream.** Synced `main` to `marimo-team/marimo` and confirmed the working branch was on the latest base before reproducing, so the bug is verified against current code, not a stale checkout.
- **`gio: http://localhost:2718: Operation not supported`.** Not an error — marimo tries to auto-open a browser, which WSL has none. Opened `http://localhost:2718` manually in the Windows browser (WSL2 forwards localhost).

Frontend was already built (`make fe` → "Nothing to be done"); backend ran via `uv run marimo edit --no-token`.

### Steps to Reproduce

The trigger condition is in the issue title: a custom CSS theme **with a background colour**. Reproduced using the wigwam theme on the repo's `kitchen_sink.py`:

1. Copy the demo notebook out of the repo (to keep the repo's git status clean):
   `cp marimo/_smoke_tests/slides_examples/kitchen_sink.py ~/repro/`
2. Download a background-colour theme into the same dir:
   `curl -sL https://raw.githubusercontent.com/Haleshot/marimo-themes/main/themes/wigwam/wigwam.css -o ~/repro/wigwam.css`
   (wigwam sets `--background: light-dark(#e6ffe9, #0b1e26)` — mint green.)
3. Attach the CSS to the notebook — either `app = marimo.App(width="medium", css_file="wigwam.css")`, or via **Notebook Settings → Custom Files → Custom CSS = `wigwam.css`**.
4. Launch from the repo (to use the editable marimo source): `uv run marimo edit ~/repro/kitchen_sink.py --no-token`, open `http://localhost:2718` in the browser. The notebook background turns mint green — confirms the CSS is applied.
5. Browser print to PDF (`Ctrl+P` → Destination: *Save as PDF*).
6. **Margins = Default** → the theme background leaves a white border around the page; the green doesn't reach the paper edge. (Matches the reporter's `margins.pdf`.)
7. **Margins = None** → the background fills the whole page, but the text sits right against the paper edge. (Matches the reporter's `no-margins.pdf`.)

**Expected:** background fills the page *and* the content keeps a comfortable inset.
**Actual:** only one or the other is achievable — there is no middle state. Both failure modes reproduce consistently and match the two PDFs attached to the issue.

### Reproduction Evidence

- Working branch: `https://github.com/lzblack/marimo/tree/fix-issue-5832`  <!-- TODO: confirm/update if branch name differs -->
- Two screenshots captured (Default-margins white border; None-margins text-against-edge), matching the issue's `margins.pdf` / `no-margins.pdf`.
