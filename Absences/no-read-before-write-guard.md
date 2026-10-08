---
type: absence
harnesses: [opencode, pi]
---
# no-read-before-write-guard

No rule that the model must `read` a file (at its current mtime) before `write`/`edit` may change it.

**What's missing**
- opencode (removed): `write` and `edit` do not require a prior read at HEAD. `edit` reads the current file inside its per-path `Semaphore(1)` before matching (`packages/opencode/src/tool/edit.ts:35-43,88,126`), but nothing tracks what the model has seen.
- pi (never had it): "No read-before-edit tracking / staleness check — nothing records reads" (observed absence, [[pi--search-replace-edit|pi]], `packages/coding-agent/src/core/tools/write.ts:67-89`).

**Evidence of decision (opencode)**
- The `FileTime` module asserted, before writes (`76a141090e^:packages/opencode/src/file/time.ts:43-99`):
  - "You must read file ${filepath} before overwriting it. Use the Read tool first" (`:92`);
  - mtime + size compare → "File … has been modified since it was last read … Please read the file again before modifying it." (`:95-99`);
  - escape hatch `OPENCODE_DISABLE_FILETIME_CHECK` (`:43`).
- `9706aaf552` (2026-01-20) "rm filetime assertions from patch tool" — first dropped for `apply_patch`.
- `76a141090e` (2026-04-16, #22999) "chore: delete filetime module" — gone everywhere. No commit body; rationale unverified.
- v2 replaces it with a narrower check: bytes read at **permission-approval** time vs at commit, under a per-path lock → "File changed after permission approval. Read it again before editing." (`packages/core/src/tool/edit.ts:112-117`; `packages/core/src/file-mutation.ts:69-82`) → [[per-file-mutation-queue]].

**Implication**
- A read-before-write rule protects against stale-clobber only if every writer goes through the harness; it also costs a full `read` round-trip per file and blocks `apply_patch` envelopes that touch files the model never read. Both harnesses settle for exact/fuzzy match failure as the staleness signal, and v2 opencode adds a compare-and-swap between approval and write.

Related: [[search-replace-edit]] · [[per-file-mutation-queue]] · [[file-op-tracking]] · [[patch-envelope-edit]] · [[pi]] · [[opencode]] · [[Absences]]
