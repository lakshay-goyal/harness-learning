---
type: concept
stage: permissions
tier: variant
aliases: [is_dangerous_command, dangerous_command_match, DangerousCommandMatch::ForcedRm, is_dangerous_command_windows, HeuristicsRuleMatch, MAX_DANGEROUS_COMMAND_WRAPPER_DEPTH, destructive-command denylist]
harnesses: [codex]
---
A small built-in denylist of obviously destructive command shapes (not an allowlist) that forces a prompt — or a hard deny when nobody can be asked — even when the sandbox/policy would otherwise auto-run the command.

## Why
- A writable-workspace sandbox happily lets `rm -rf .` delete the user's uncommitted work; the kernel boundary protects the system, not the project.
- Wrappers (`sudo`, `env`, `bash -lc`) hide the dangerous verb from naive matching.
- "Never ask" must not mean "allow destructive" ([[dangerous-command-under-never]]).

## Design space
- **Regex list in an opt-in extension** — pi example `permission-gate.ts` (`rm -rf`, `sudo`, `chmod 777`) ([[pi--tool-call-gate]]).
- **Tiny parsed denylist in core** — ✔ codex: POSIX forced `rm`, unwrapping `sudo`/`env`/`trap`, nested `-lc` scripts; Windows `del /f`, `Remove-Item -Force`, ShellExecute launchers.
- **Large curated allowlist of safe commands** — codex had, deleted ([[no-safe-command-allowlist]]).
- **Unknown/too-deep wrappers**: treat as dangerous (✔ codex depth 8).
- **Outcome**: prompt / forbid under no-prompt modes (✔ codex) vs. warn only.
- **Complement**: prompt-level rules ("NEVER use `git reset --hard`…") — [[destructive-git-on-user-changes]]; LLM reviewer policy for destructive actions ([[llm-approval-reviewer]]).

## Implementations
- [[codex--dangerous-command-heuristics|codex]] — `is_dangerous_command` (forced `rm`, wrappers, depth 8) + `windows_dangerous_commands`; hit → Prompt, Forbidden under `Never`.

## Failures
- [[dangerous-command-under-never]]
- [[destructive-git-on-user-changes]]

## Related
[[command-rule-policy]] · [[approval-policy-modes]] · [[shell-command-intent-parsing]] · [[os-level-sandbox]] · [[tool-safety-annotations]] · [[no-safe-command-allowlist]]
