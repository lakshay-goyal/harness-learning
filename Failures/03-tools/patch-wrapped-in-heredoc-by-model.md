---
type: failure
concepts: [patch-envelope-edit, tool-argument-repair]
harnesses: [codex]
---
**Symptom** — GPT-4.1 sent `apply_patch` input wrapped in a shell heredoc (`<<'EOF' … EOF`) — a form that only works when the patch is piped through bash, not when passed as the tool argument — so the patch was unparseable. "GPT-4.1 does not always generate a valid invocation of `apply_patch`. Fortunately, the error is predictable" (`6fcc528a43` body).

**Root cause** — The model was trained on the shell form (`apply_patch <<'EOF'`) and reproduces the wrapper even when calling the tool directly.

**Fix · [[codex]]**
- `6fcc528a43` 2025-06-03 "provide tolerance for apply_patch tool (#993)" — `ParseMode::Lenient` strips `<<EOF` / `<<'EOF'` / `<<"EOF"` wrappers (`codex-rs/apply-patch/src/parser.rs:165-190,225-252`).
- `PARSE_IN_STRICT_MODE = false`: lenient for **all** models because threading a per-model strictness flag "is a pain" (`parser.rs:47-53`).
- Later prevention upstream: freeform grammar-constrained tool (`4764fc1ee7` 2025-10-04, default `cce059467a` 2026-05-08) → [[malformed-tool-json-crashes]].

**Lesson** — When a model reliably makes the same envelope mistake, strip it at parse time rather than fight it in the prompt.

Related: [[patch-envelope-edit]] · [[tool-argument-repair]] · [[codex--patch-envelope-edit|codex]] · [[tool-name-training-artifact]] · [[patch-body-executed-as-shell]]
