---
type: implementation
harness: codex
concept: command-rule-policy
commit: 622e9e3696
files: [codex-rs/execpolicy/README.md:6, codex-rs/execpolicy/src/parser.rs:117, codex-rs/execpolicy/src/parser.rs:349, codex-rs/execpolicy/src/policy.rs:403, codex-rs/execpolicy/src/amend.rs:65, codex-rs/core/src/exec_policy.rs:48, codex-rs/core/src/exec_policy.rs:57, codex-rs/core/src/exec_policy.rs:662, codex-rs/core/src/exec_policy.rs:770, codex-rs/core/src/exec_policy.rs:872, codex-rs/core/src/exec_policy.rs:958]
---
[[command-rule-policy]] in [[codex]].

## Mechanism
- Language: Starlark (`starlark` crate) with builtins `prefix_rule(pattern, decision?, justification?, match?, not_match?)`, `network_rule(host, protocol, decision, justification?)`, `host_executable(name, paths)` (`codex-rs/execpolicy/README.md`; `codex-rs/execpolicy/src/parser.rs:349`, `:410-435`, `:437`). Pattern tokens match in order; a list element = alternatives. `decision` defaults `allow`; `"deny"` parses as `Forbidden` (`codex-rs/execpolicy/src/parser.rs:255`).
- `match` / `not_match` examples validated at load — "think of them as unit tests" (`codex-rs/execpolicy/src/parser.rs:117-150`). README: forbidden rules should carry an alternative in `justification` ("Use `jj` instead of `git`.") (`codex-rs/execpolicy/README.md:8`).
- Effective decision = strictest across all matches, `forbidden > prompt > allow` (`codex-rs/execpolicy/src/policy.rs:403`; `codex-rs/execpolicy/README.md:95`).
- Executable identity: basename fallback (`/usr/bin/git` → `git` rules) only when host-executable resolution is on and, if `host_executable(name=…)` exists, only for its listed absolute paths (README; `b148d98e0e`). `codex-rs/core/src/exec_policy/executable_identity.rs` (not read in detail).
- Loading (`load_exec_policy`, `codex-rs/core/src/exec_policy.rs:662-716`): every config layer low→high contributes `<layer>/rules/*.rules` (`RULES_DIR_NAME="rules"`, `RULE_EXTENSION="rules"`, `DEFAULT_POLICY_FILE="default.rules"`, `:54-56`); "Disabled project layers already represent the trust decision" (`:663-664`); admin requirements policy merged as overlay and may ignore user/project rules entirely.
- Compound scripts: `bash -lc "a && b"` parsed into plain commands, each evaluated (`codex-rs/core/src/exec_policy.rs:872-905`); PowerShell parsed on Windows. Prompt tells the model advanced shell features (redirection, substitution, env vars, globs) "will not be evaluated against rules" ([[codex--permission-state-prompt]]).
- Unmatched commands (`render_decision_for_unmatched_command_for_platform`, `codex-rs/core/src/exec_policy.rs:770-855`): dangerous-heuristic hit → Prompt (Forbidden under Never); Never → Allow ("relying on the sandbox for protection"); UnlessTrusted → Prompt; OnRequest + restricted FS → Allow unless the model asked for a sandbox override; OnRequest + unrestricted → Allow. Windows with sandbox disabled but managed FS restrictions → dangerous ("there is no platform sandbox to enforce the boundary").
- Conflict reasons (`codex-rs/core/src/exec_policy.rs:48-53`): `PROMPT_CONFLICT_REASON` "approval required by policy, but AskForApproval is set to Never"; `REJECT_SANDBOX_APPROVAL_REASON` (Granular.sandbox_approval false); `REJECT_RULES_APPROVAL_REASON` (Granular.rules false).
- Amendments: approved-with-amendment appends `prefix_rule(..., decision="allow")` to `default.rules` under an advisory file lock (`codex-rs/execpolicy/src/amend.rs:65`, `:147-160`); network approvals persisted as `network_rule` (`codex-rs/execpolicy/src/amend.rs:85`; `c3048ff90a`). Newly saved prefixes/rules are announced to the model as small developer notes ("Approved command prefix saved:", `codex-rs/core/src/context/approved_command_prefix_saved.rs:5`).
- Model-proposed `prefix_rule` guardrails (`derive_requested_execpolicy_amendment_from_prefix_rule`, `codex-rs/core/src/exec_policy.rs:958-997`): rejected if equal to any of 88 `BANNED_PREFIX_SUGGESTIONS` (`/bin/bash -lc`, `bash`, `sh -c`, `zsh -lc`, `cmd /c`, `powershell -EncodedCommand`, `python`, `python3`, `node -e`, `perl -e`, `ruby`, `deno eval`, `bun run`, `npm run`, `pnpm run`, `yarn run`, `env`, `git`, `sudo`, `rm`, `osascript`, `Rscript`, `julia -e`, `lua -e`, `php -r` …; `codex-rs/core/src/exec_policy.rs:57-146`); rejected if an existing rule already matched; accepted only if adding it would allow every parsed sub-command (`prefix_rule_would_approve_all_commands`, `codex-rs/core/src/exec_policy.rs:999-1021`).
- CLI: `codex execpolicy check` (README). `docs/execpolicy.md` is a 3-line stub (`docs/execpolicy.md:1-3`).

## Constants
| name | value | path:line |
|---|---|---|
| rules dir / ext / default file | `rules` / `.rules` / `default.rules` | `codex-rs/core/src/exec_policy.rs:54-56` |
| banned prefix suggestions | 88 entries | `codex-rs/core/src/exec_policy.rs:57-146` |
| decision order | forbidden > prompt > allow | `codex-rs/execpolicy/src/policy.rs:403` |

## Evolution
- 2025-04-24 `58f0e5ab74` "introduce codex_execpolicy crate for defining 'safe' commands (#634)" — typed-argument Starlark language replacing TS prefix matching.
- 2025-07-21 `6cf4b96f9d` ripgrep flags checked before trusting (safe-list maintenance era).
- 2025-11-17 `a941ae7632` "execpolicy v2 (#6467)"; 2025-11-19 `fb9849e1e3` execpolicy → execpolicy-legacy, execpolicy2 → execpolicy (prefix-rule-only).
- 2025-12-10 `c4af707e09` removed experimental "command risk assessment".
- 2025-12-11 `e0d7ac51d3` `policy/*.codexpolicy` → `rules/*.rules`.
- 2026-01-05 `cafb07fe6e` `justification` arg.
- 2026-01-28 `996e09ca24` model-proposed `prefix_rule` ("RequestRule").
- 2026-02-12 `e6e4c5fa3a` "Restrict model-suggested rules (#11671)".
- 2026-02-23 `c3048ff90a` persist network approvals in execpolicy.
- 2026-02-27 `b148d98e0e` `host_executable()` path mappings.
- 2026-07-10 `656a2d0905` "Remove the legacy exec policy engine (#32093)"; 2026-07-20 `bf3c1972b7` migrate legacy allow rules.
- 2026-08-16 `f85e81d30b` requirements policy ownership moved to execpolicy.
- 2026-08-19 `942af8447b` known-safe allowlist deleted; rules + sandbox carry auto-run decisions.

## Quirks
- README: "a richer language will follow" (`codex-rs/execpolicy/README.md:6`) — the richer language came first (legacy) and was deleted.
- An `allow` decides *whether to run*, never *whether to drop the sandbox* when deny-reads exist ([[escalation-drops-deny-read]]).

## Versus pi
- pi has no rules file; [[pi--tool-call-gate]] example `permission-gate.ts` hard-codes a regex list in TypeScript.
