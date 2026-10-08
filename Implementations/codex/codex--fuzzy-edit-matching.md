---
type: implementation
harness: codex
concept: fuzzy-edit-matching
commit: 622e9e3696
files: [codex-rs/apply-patch/src/seek_sequence.rs:1-110, codex-rs/apply-patch/src/file_update.rs:68-156]
---
[[fuzzy-edit-matching]] in [[codex]] — applied to `apply_patch` context/old lines ([[patch-envelope-edit]]), not to search/replace anchors.

## Mechanism
- `seek_sequence(lines, pattern, start, eof)` (`codex-rs/apply-patch/src/seek_sequence.rs:1-110`): line-wise, whole-line comparison in **4 passes of decreasing strictness**, first hit wins:
  1. exact;
  2. `trim_end` equal (trailing whitespace ignored);
  3. `trim` equal (leading + trailing whitespace ignored);
  4. `normalise`: trim + Unicode punctuation → ASCII — dashes U+2010–2015 / U+2212 → `-`; single quotes U+2018–201B → `'`; double quotes U+201C–201F → `"`; NBSP U+00A0, U+2002–200A, U+202F, U+205F, U+3000 → space (`:66-110`). Comment: "This mirrors the fuzzy behaviour of `git apply` which ignores minor byte-level differences when locating context lines."
- `eof` chunks (`*** End of File`) first try matching at end of file, then fall back to searching from `start` (`:4-6,29-34`).
- Defensive: empty pattern → `Some(start)`; pattern longer than file → `None` ("avoids out‑of‑bounds panic that occurred pre‑2025‑04‑12") (`:8-28`).
- `@@ <ctx>` anchor line is located with the same seek, then the old-lines search continues after it (`codex-rs/apply-patch/src/file_update.rs:68-84`); retry without the trailing empty sentinel line for EOF modifications (`file_update.rs:94-120`).
- **Write side**: the replaced region is written from the patch's `+` lines, so whitespace/typography inside the replacement comes from the model, not the file; untouched lines keep their bytes and (since `685270a56a`) their original line endings → [[edit-rewrites-line-endings]].
- No uniqueness check: first match at/after the cursor wins (context lines + `@@` anchors disambiguate) — contrast pi's uniqueness requirement ([[search-replace-edit]]).
- Errors to model (via "apply_patch verification failed: …"): "Failed to find context '{ctx}' in {path}" / "Failed to find expected lines in {path}:\n{lines}" (`file_update.rs:81,156`).
- No NFKC, no invisible-character (BOM, zero-width) normalization; the model is not told a fuzzy pass matched.

## Evolution
- 2025-04-24 `31d0d7a305` exact / trim_end / trim passes with the Rust import (pattern-length panic fix dated "pre-2025-04-12" in comment, TS era).
- 2025-04-25 `15bf5ca971` "handling weird unicode characters in `apply_patch`" — 4th normalization pass ([[edit-invisible-character-mismatch]]).

## Versus pi
pi: exact → normalized (NFKC, trailing ws, quotes, dashes, unicode spaces) on whole-file text with uniqueness counted in normalized space, then line-preserving overlay write ([[pi--fuzzy-edit-matching]]). codex: line-granular passes, no NFKC, no uniqueness, replacement text taken verbatim from the patch.
