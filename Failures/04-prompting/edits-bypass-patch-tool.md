---
type: failure
concepts: [guideline-softening, dynamic-tool-guidelines]
harnesses: [codex]
---
**Symptom** — GPT-5-codex made ~22% of file edits "using something else than a raw `apply_patch`" (Python scripts, `cat`, `sed`, `["bash","-lc","apply_patch"]`) — `f6a152848a` body; even after the fix "a few `["bash", "-lc", "apply_path"]` when reaching < 10% context left".

**Root cause** — Shell is always available and the model's training mixes edit styles; without a stated rule it picks whatever is at hand, bypassing the edit tool's verification, approval and diff tracking.

**Fix · [[codex]]**
- `f6a152848a` 2025-09-30 "When editing or creating files, you MUST use apply_patch as a standalone tool without going through ["bash", "-lc"], `Python`, `cat`, `sed`, ..." → measured 0% bypass.
- Reverted next day `5d78c1edd3` 2025-10-01 (no reason given).
- `0ad1b0782b` 2025-10-04 permissive rule with explicit carve-outs: "Try to use apply_patch for single file edits, but it is fine to explore other options to make the edit if it does not work well. Do not use apply_patch for changes that are auto-generated (i.e. generating package.json or running a lint or format command like gofmt) or when scripting is more efficient (such as search and replacing a string across a codebase)." (`codex-rs/core/gpt_5_codex_prompt.md:11`).
- Same week `4764fc1ee7` 2025-10-04 made apply_patch a FREEFORM tool ("This is a FREEFORM tool, so do not wrap the patch in JSON.") → [[patch-envelope-edit]]. Today's catalog text: "Use `apply_patch` for manual code edits. Do not create or edit files with `cat` or other shell write tricks" (`codex-rs/models-manager/models.json:1408`).

**Lesson** — An absolute MUST can hit 100% compliance in eval and still be reverted; the shipped form names when the alternative is legitimate. (Codex analogue of [[shell-cat-instead-of-read-tool]].)

Related: [[guideline-softening]] · [[dynamic-tool-guidelines]] · [[patch-envelope-edit]] · [[shell-cat-instead-of-read-tool]] · [[codex--guideline-softening|codex]]
