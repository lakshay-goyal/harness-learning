---
type: absence
harnesses: [codex]
---
# no-in-repo-user-docs

User documentation lives off-repo (developers.openai.com); in-repo docs are pointers.

**What's missing**
- All of `docs/*.md` total ~206 lines and mostly link out (e.g. `docs/sandbox.md` → developers.openai.com/codex/security; `docs/config.md` pointers + one note on `allow_managed_hooks_only`); `SECURITY.md` → "Agent approvals & security"; `CHANGELOG.md` → GitHub releases.

**Evidence of decision**
- Observed at `622e9e3696` (M8 repo facts).

**Implication**
- The model cannot be pointed at bundled harness docs; contrast pi's [[self-documentation-pointer]]. Only in-repo AGENTS.md is `codex-rs/tui/src/bottom_pane/AGENTS.md`.

Related: [[self-documentation-pointer]] · [[layered-settings]] · [[Absences]]
