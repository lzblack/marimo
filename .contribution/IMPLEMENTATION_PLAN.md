# Implementation Plan — marimo #5832 (UMPIRE)

## Understand

A marimo notebook using a custom CSS theme **with a background colour** can't be exported to a good-looking PDF. With the browser's default print margins, the background leaves a white border around the page; with zero margins, the background fills the page but the text is pressed against the paper edge. There is no middle state where the background fills the page *and* the content keeps a comfortable inset. The desired behaviour: background fills the page, content has padding, and that padding scales with marimo's width config (`compact` / `medium` / `full`). This is an **enhancement** to the in-app "Download as PDF" (browser-print) path.

## Match

The existing print styles live in `frontend/src/css/app/print.css` (and a smaller `@media print` block in `frontend/src/css/md.css`). Reading the current `print.css`:

- `@media print { * { print-color-adjust: exact; ... } }` — background printing is **already implemented** (this is why the green background prints even without the browser's "Background graphics" toggle). No change needed here.
- There is **no `@page` rule anywhere** — so marimo currently doesn't control page margins at all; they're left entirely to the browser. This is the root of the "white border" half of the problem.
- `.output-area` / `.console-output-area` are explicitly zeroed: `padding-left: 0 !important; padding-right: 0 !important;`. This is the direct cause of the "text against the edge" half.

So the file already establishes the pattern (a print-specific `@media print` block) that the fix extends.

## Plan

Root cause: in print CSS, marimo neither sets a `@page` margin nor gives the content any inset, so margin is fully delegated to the browser — which produces either a white border or edge-to-edge text. Move the spacing from the page layer to the content layer:

1. Add `@page { margin: 0 }` so the background reaches the paper edge (eliminate the white border).
2. Replace `.output-area`'s `padding: 0` with a meaningful print padding so text keeps an inset (eliminate edge-to-edge text).
3. Make that padding scale with the width config (`compact` / `medium` / `full`).
4. `print_background` (`print-color-adjust: exact`) is already present — leave it.

**File(s):** `frontend/src/css/app/print.css` (possibly the `@media print` block in `md.css`).

**Scope:**

- *In scope:* the in-app "Download as PDF" (browser-print) path — the reporter's original target and the path `print.css` governs.
- *Out of scope (to state in the PR):* the `marimo export pdf` CLI/UI path. On current `main` that path goes through nbconvert (PDFExporter/LaTeX, fallback WebPDFExporter) and does **not** use `print.css`, so it would be a separate change. (`prefer_css_page_size` survives only in the reveal.js *slides* export path, not regular notebook export.)

## Implement

(Phase III.) Working branch: `https://github.com/lzblack/marimo/tree/fix-issue-5832`  <!-- TODO: confirm -->

## Review

Self-review against `CONTRIBUTING.md` before opening the PR:

- Lint/typecheck/format must pass: `make fe-check` (or `make check`).
- Code changes should be accompanied by tests (CONTRIBUTING requirement) — see Evaluate for the CSS-testing question.
- CLA: first PR fails CI until signed; sign by commenting the CLA text on the PR.
- Maintainer approval already obtained (Myles welcomed it + `help wanted`; Light2Dark reviewed the approach).
- Follow the repo's commit-message / PR conventions.

## Evaluate

- **Manual:** re-run the reproduction (wigwam + `kitchen_sink.py`), export to PDF, and confirm the background fills the page *and* the text keeps an inset — across `compact` / `medium` / `full` widths, and against both Default and None browser margins. Visually compare against the issue's `margins.pdf` / `no-margins.pdf` and validate on `kitchen_sink.py` as Light2Dark requested.
- **Automated:** a pure print-CSS change is awkward to unit-test; decide between an e2e/print snapshot or asserting the emitted styles. Confirm what kind of test the maintainers expect for a CSS-only change before assuming.

### Open items to verify in Phase III

1. **In-app "Download as PDF" trigger mechanism.** If it opens the browser print *dialog* (user picks margins), the dialog's margin choice overrides `@page margin`, so the reliable fix is the content padding (white-border half may be limited). If it prints programmatically (honouring `@page`), `@page margin: 0` takes effect too. Verify by testing the actual in-app button (not manual `Ctrl+P`) after installing `nbconvert[webpdf]`.
2. **Width-config selector.** Locate how `compact` / `medium` / `full` map to the DOM (class vs data attribute) so the padding can scale:
   `grep -rniE "data-width|width-(compact|medium|full)|appWidth" frontend/src | grep -v node_modules`
3. **Export (nbconvert) path** confirmed out of scope for this PR.
