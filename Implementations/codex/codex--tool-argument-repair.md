---
type: implementation
harness: codex
concept: tool-argument-repair
commit: 622e9e3696
files: [codex-rs/apply-patch/src/parser.rs:47-53, codex-rs/apply-patch/src/parser.rs:165-252, codex-rs/apply-patch/src/invocation.rs:27, codex-rs/apply-patch/src/invocation.rs:141-157, codex-rs/core/src/unified_exec/mod.rs:218-226, codex-rs/core/src/tools/handlers/view_image.rs:133-142, codex-rs/core/src/tools/registry.rs:653-655, codex-rs/core/src/tools/code_mode/wait_spec.rs:53]
---
[[tool-argument-repair]] in [[codex]] — repair concentrated on the one free-text tool (`apply_patch`); JSON tools mostly clamp or reject.

## Mechanism
- **Patch envelope repair** ([[patch-envelope-edit]]):
  - Lenient parse for every model (`PARSE_IN_STRICT_MODE = false`) strips `<<EOF` / `<<'EOF'` / `<<"EOF"` heredoc wrappers that GPT-4.1 put around patches (`codex-rs/apply-patch/src/parser.rs:47-53,165-190,225-252`; `6fcc528a43`) → [[patch-wrapped-in-heredoc-by-model]].
  - Misspelled command name `applypatch` accepted alongside `apply_patch` in shell interception and arg0 dispatch (`codex-rs/apply-patch/src/invocation.rs:27,309`; `5f8984aa7d`: "silently handle this case to avoid hurting model performance"; heredoc path `1a1516a80b`) → [[tool-name-training-artifact]].
  - `exec_command` carrying `apply_patch <<'EOF'…` is re-routed to the patch tool instead of failing ([[shell-execution]]).
  - **Not** repaired: a raw patch passed as a shell command/script → explicit error `patch detected without explicit call to apply_patch. Rerun as ["apply_patch", "<patch>"]` (`invocation.rs:141-157`) — the error names the exact fix ([[patch-body-executed-as-shell]]).
- **Clamp instead of reject** for out-of-range waits: `yield_time_ms` clamped to [250, 30000] (Windows floor 10000) (`codex-rs/core/src/unified_exec/mod.rs:218-226`); `wait_agent` timeouts below the floor clamped to the configured minimum (`4d7e3e90d9` 2026-08-07 "Clamp short wait_agent timeouts to the configured minimum"; floor `codex-rs/core/src/config/mod.rs:259`) → [[task-owned-subagent]].
- **Legacy values tolerated**: `view_image.detail` "Keep accepting previously supported detail hints after they disappear from the schema" (`high` / `original`), else "view_image.detail only supports `high` or `original`; omit `detail` for default high resized behavior, got `{detail}`" (`codex-rs/core/src/tools/handlers/view_image.rs:133-142`).
- **Reject with bounds stated**: `clock.sleep` `deny_unknown_fields` + "duration_ms must be between 1 and 43200000" ([[wall-clock-tools]]); `tool_search` limit 0 rejected ([[deferred-tool-loading]]); bad JSON → "failed to parse function arguments: {err}" ([[tool-error-as-result]]).
- **Hook-based rewrite**: PreToolUse hooks may return `updated_input`, re-validated via `with_updated_hook_input` (`codex-rs/core/src/tools/registry.rs:653-655`) → [[tool-call-gate]].
- **Client completes the shape**: `request_user_input` options must not include "Other"; the client adds it (`is_other`) ([[structured-user-question-tool]]).
- **Upstream prevention** instead of repair: grammar-constrained freeform `apply_patch` (`4764fc1ee7`) and foreign-schema sanitization ([[constrained-tool-sampling]], [[tool-schema-normalization]]).
- Catalog-side (not model args): invalid model-catalog tool parameter overrides fall back to bundled parameters with "Invalid catalog tool parameters; using bundled parameters" (`codex-rs/core/src/tools/code_mode/wait_spec.rs:53`).

## Evolution
- 2025-06-03 `6fcc528a43` lenient patch parsing. 2025-08-11 `5f8984aa7d` / 2025-08-20 `1a1516a80b` `applypatch` alias. 2025-09-12 `5f6e95b592` implicit invocation error.
- 2025-10-04 `4764fc1ee7` freeform grammar removes JSON escaping.
- 2026-01-14 `32b1795ff4` clamp empty-poll yield. 2026-08-07 `4d7e3e90d9` clamp wait_agent.

## Versus pi
pi repairs JSON argument *shapes* per tool (`prepareArguments`: stringified edits, single object, legacy field names) and coerces types generically; it rejects out-of-range timeouts ([[pi--tool-argument-repair]]). codex repairs the *patch envelope* and clamps waits; its JSON schemas are simple enough that shape drift isn't a recorded problem.
