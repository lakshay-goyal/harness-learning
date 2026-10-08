---
type: failure
concepts: [patch-envelope-edit, tool-argument-repair]
harnesses: [codex]
---
**Symptom** — GPT-OSS and gpt-5-mini "occasionally use `applypatch` instead of `apply_patch`" in shell commands (`5f8984aa7d` body); the misspelled command failed. Prompt-side, codex already had to say "NEVER try `applypatch` or `apply-patch`, only `apply_patch`" (`81b148bda2`).

**Root cause** — A tool name the model family was trained on (with variants) is a learned token sequence; the prompt rule alone did not stop it.

**Fix · [[codex]]**
- `5f8984aa7d` 2025-08-11 "[apply-patch] Support applypatch command string (#2186)" — accept the alias in shell interception: "silently handle this case to avoid hurting model performance"; `APPLY_PATCH_COMMANDS = ["apply_patch", "applypatch"]` (`codex-rs/apply-patch/src/invocation.rs:27,309`).
- `1a1516a80b` 2025-08-20 heredoc form of `applypatch` fixed.
- arg0 dispatch also maps `applypatch` to the bundled patch binary (`codex-rs/arg0/src/lib.rs:21-25`).

**Lesson** — Alias the misspellings your model family was trained on instead of rejecting them; it is the owning-harness side of [[foreign-harness-tool-hallucination]].

Related: [[patch-envelope-edit]] · [[tool-argument-repair]] · [[foreign-harness-tool-hallucination]] · [[codex--patch-envelope-edit|codex]] · [[codex--tool-argument-repair|codex repair]]
