---
type: concept
stage: permissions
tier: candidate
aliases: [execpolicy, codex-execpolicy, prefix_rule, network_rule, host_executable, "*.rules", default.rules, Decision::Forbidden, ExecPolicyAmendment, ApprovedExecpolicyAmendment, codex execpolicy check, BANNED_PREFIX_SUGGESTIONS, execpolicy-legacy, command allow/deny rules]
harnesses: [codex]
---
A declarative, user/admin-editable rules file that matches argv prefixes of shell commands (and network hosts) to allow / prompt / forbidden, evaluated before execution with the strictest matching rule winning; user approvals are appended as new rules so the policy learns.

## Why
- Session-only approvals re-prompt for `git pull` every day; persistent rules turn approvals into durable, reviewable config.
- Admins need a non-overridable floor ("never `git push --force`") above whatever the user or model approves.
- Matching on raw strings is unsafe: `bash -lc "a && b"` must be split into its commands; a rule for `git` must not approve `/tmp/evil/git`.
- Once the model can draft its own rules, the rule format becomes an escalation channel ([[model-proposed-rule-too-broad]]).

## Design space
- **None** — pi (no built-in allow/deny config; only regex lists inside example extensions, [[pi--tool-call-gate]]).
- **Curated built-in safe-command allowlist** — codex had `is_known_safe_command` (+ Windows), deleted 2026-08-19 ([[no-safe-command-allowlist]]).
- **Typed argument language** (codex legacy Starlark execpolicy with arg types, 2025-04) vs **prefix-rule subset** (codex execpolicy v2, 2025-11; legacy deleted 2026-07).
- **Combination**: strictest wins (✔ codex `forbidden > prompt > allow`) vs first match vs last layer wins.
- **Layering**: per config layer `rules/` dirs + admin requirements overlay that may ignore user/project rules (✔ codex); project rules gated by trust ([[project-trust-gate]]).
- **Learning**: approvals appended as rules (✔ codex `ApprovedExecpolicyAmendment`, `NetworkPolicyAmendment`) vs session cache only.
- **Who drafts the rule**: mechanical derivation vs model-proposed `prefix_rule` validated against a ban list (✔ codex).
- **Executable identity**: basename match vs absolute path mapping (`host_executable`, ✔ codex).
- **Self-tests in the policy** (`match`/`not_match` examples validated at load, ✔ codex).
- **Unmatched commands**: fall back to approval mode + sandbox + [[dangerous-command-heuristics]] (✔ codex).

## Implementations
- [[codex--command-rule-policy|codex]] — Starlark `prefix_rule` / `network_rule` / `host_executable` in `<layer>/rules/*.rules`; strictest wins; amendments appended to `default.rules` under lock; ~88 banned model-suggested prefixes; compound scripts split.

## Failures
- [[model-proposed-rule-too-broad]]
- [[escalation-drops-deny-read]]
- [[dangerous-command-under-never]]

## Related
[[approval-policy-modes]] · [[sandbox-escalation-retry]] · [[dangerous-command-heuristics]] · [[approval-key-canonicalization]] · [[egress-policy-proxy]] · [[layered-settings]] · [[project-trust-gate]] · [[shell-command-intent-parsing]] · [[permission-state-prompt]] · [[tool-call-gate]] · [[no-safe-command-allowlist]]
