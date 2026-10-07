---
type: implementation
harness: codex
concept: project-trust-gate
commit: 622e9e3696
files: [codex-rs/config/src/config_toml.rs:604, codex-rs/config/src/loader/mod.rs:1085, codex-rs/config/src/loader/mod.rs:1093, codex-rs/core/src/agents_md.rs:64, codex-rs/core/src/config/mod.rs:3753, codex-rs/core/src/config/permissions.rs:50, codex-rs/core/src/exec_policy.rs:663]
---
[[project-trust-gate]] in [[codex]].

## Mechanism
- Storage: per-project entry in user config, `[projects."<path>"] trust_level = "trusted" | "untrusted"` (`ProjectConfig{trust_level}`, `codex-rs/config/src/config_toml.rs:604-617`). Trust key = repo root; linked worktrees inherit (`codex-rs/config/src/loader/mod.rs:1085-1090`). Worktree roots resolved by inspecting `.git` entries on disk, not by running `git rev-parse`, so repo-controlled git config never executes pre-trust (`5deaf9409b`).
- What untrusted disables: the project config layer is still loaded but carries a `disabled_reason` naming "project-local config, hooks, and exec policies" (`codex-rs/config/src/loader/mod.rs:1093-1105`); exec-policy loading reuses that view ("Disabled project layers already represent the trust decision", `codex-rs/core/src/exec_policy.rs:663-664`); project `AGENTS.md` skipped when untrusted, trust level in the instruction cache key (`codex-rs/core/src/agents_md.rs:64-66`; `bd19459358`). PATH helpers in the workspace are not executed before trust (`637c3227b3`).
- Trust also picks **runtime defaults** (beyond an input-loading gate): trusted → approval `OnRequest`; explicitly untrusted → internal `UnlessTrusted` (approval for every command not allowed by a rule); undecided → default (`codex-rs/core/src/config/mod.rs:3759-3770`). Default permission profile: workspace-write if a decision exists (trusted or untrusted), read-only if never decided or Windows sandbox disabled (`codex-rs/core/src/config/permissions.rs:50-61`). Explicit `approval_policy = "untrusted"` is a config error (`codex-rs/core/src/config/mod.rs:3753-3758`).
- Decision UX: undecided local projects prompt; auto-trust was tried and reverted the same day (`1e59dc5bda` → `17801b4206`: "Trusting a directory enables project-local config, hooks, and exec policies, which can increase exposure to prompt injection"). Projectless dirs don't persist trust (`608e4cc9a1`).
- Second trust layer for hooks: unmanaged hooks run only if their normalized definition hash matches a persisted `trusted_hash`; managed (admin) hooks always runnable (`0452dca986`; review in `codex-rs/hooks/src/engine/discovery.rs`). Bypass flag `--dangerously-bypass-hook-trust` ("DANGEROUS. Intended only for automation that already vets hook sources", `codex-rs/utils/cli/src/shared_options.rs:60-63`; `392e94e9ea`).
- Writes into project `.codex/` / `.agents/` by the agent are blocked by the sandbox so a trusted project cannot be silently extended by the agent ([[codex--protected-workspace-metadata]]).
- Marketplace manifests in repos cannot claim reserved marketplace names (`0acf302db5`).

## Evolution
- 2025-06-24/25 `86d5a9d80d` / `72082164c1` unless-allow-listed → `untrusted` / `UnlessTrusted`.
- 2025-08-21 `750ca9e21d` explicit `[projects]` tables; 2025-08-22 `6f0b499594` detect git worktrees for project trust.
- 2026-03-06 `5deaf9409b` "avoid invoking git before project trust is established".
- 2026-05-05 `0452dca986` hook trust metadata and enforcement; 2026-05-13 `392e94e9ea` `--dangerously-bypass-hook-trust`.
- 2026-08-04 `1e59dc5bda` auto-trust undecided projects → same day `17801b4206` prompt instead.
- 2026-08-18 `0acf302db5` marketplace identity spoofing blocked.
- 2026-08-19 `942af8447b` "Retire the untrusted approval policy" — user-facing mode removed, untrusted projects prompt for every command.
- 2026-08-21 `bd19459358` "Ignore project instructions for untrusted projects (#39837)".
- 2026-09-02 `637c3227b3` "Avoid executing PATH helpers before workspace trust".
- 2026-09-18 `608e4cc9a1` no trust persisted for projectless directories.

## Versus pi
- Same core idea as [[pi--project-trust-gate]] (gate repo config/plugins), but codex **gates AGENTS.md** (pi ungated it as unpreventable injection) and lets trust choose approval + sandbox defaults; pi offers parent/session scopes and plugin-decidable trust, codex keys on repo root.
