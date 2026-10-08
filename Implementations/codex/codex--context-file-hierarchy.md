---
type: implementation
harness: codex
concept: context-file-hierarchy
commit: 622e9e3696
files: [codex-rs/core/src/agents_md.rs:1, codex-rs/core/src/agents_md.rs:43, codex-rs/core/src/agents_md.rs:54, codex-rs/core/src/agents_md.rs:63, codex-rs/core/src/agents_md.rs:150, codex-rs/core/src/agents_md.rs:200, codex-rs/core/src/agents_md.rs:272, codex-rs/core/src/agents_md.rs:386, codex-rs/core/src/agents_md_manager.rs:186, codex-rs/core/src/context/user_instructions.rs, codex-rs/core/src/context/world_state/agents_md.rs:11, codex-rs/config/src/config_toml.rs:75, codex-rs/protocol/src/prompts/base_instructions/default.md:17, codex-rs/codex-home/src/instructions/mod.rs:13]
---
[[context-file-hierarchy]] in [[codex]].

## Mechanism
- **Discovery** (`codex-rs/core/src/agents_md.rs:1-18`): walk up from cwd to the nearest `project_root_markers` ancestor (default `.git`, `:10-14`; unset → default; empty list disables parent traversal); collect root→cwd inclusive; never walk past root; no marker found → only cwd. Per directory the first existing of `AGENTS.override.md`, `AGENTS.md`, then configured `project_doc_fallback_filenames` wins — one file per directory (`:43-45`, `:272-297`). Probes run concurrently, `MAX_CONCURRENT_ANCESTOR_PROBES = 256` (`:54`).
- `project_root_markers` comes from merged config layers EXCLUDING project-layer config (`codex-rs/core/src/agents_md.rs:200-205`).
- Fallback names containing path syntax (`.`, `..`, `/`, NUL; on Windows executors also `\` and `:`) are dropped before probing: "Probing Windows network paths can send ambient credentials, even during metadata checks" (`50d77959bf` body; check at `codex-rs/core/src/agents_md.rs:281-292`).
- **Global/user level**: `AGENTS.md` / `AGENTS.override.md` loaded from `CODEX_HOME` (`codex-rs/codex-home/src/instructions/mod.rs:12-13`).
- **Trust gate**: untrusted project → project AGENTS.md skipped entirely, user-level instructions kept (`codex-rs/core/src/agents_md.rs:63-65`); trust level is part of the instruction cache key. Reads go through the environment's filesystem sandbox; a sandbox-blocked discovered file fails thread/turn setup (`7ece061767`).
- **Budget**: `project_doc_max_bytes` default 32768 (`codex-rs/config/src/config_toml.rs:75`, `codex-rs/config/defaults.toml:8`; `AGENTS_MD_MAX_BYTES` `codex-rs/core/src/config/mod.rs:256`). One shared budget across all files and environments, consumed root→cwd; the crossing file is byte-truncated (UTF-8 lossy), later files dropped; `tracing::warn!` "project doc exceeds remaining budget; truncating" — NOT told to the model (`codex-rs/core/src/agents_md.rs:150-163`).
- **Host/thread instructions**: `MAX_THREAD_INSTRUCTIONS_TOKENS = 10_000` estimated tokens; oversized rejected, not truncated (`codex-rs/core/src/agents_md_manager.rs:186-200`).
- **Rendering**: one `user`-role message `# AGENTS.md instructions for <cwd>\n\n<INSTRUCTIONS>\n…\n</INSTRUCTIONS>` (`codex-rs/core/src/context/user_instructions.rs`, markers `("# AGENTS.md instructions", "</INSTRUCTIONS>")`, kind `agents_md.instructions`). Inside: user/global instructions first, then project files joined by `\n\n`; `\n\n--- project-doc ---\n\n` only once at the user→project transition (`codex-rs/core/src/agents_md.rs:49`, `:386-419`). No per-file path label; multi-environment groups labeled "for `<env>` with root <path>" (`:421-470`).
- **Prompt-taught semantics** (`codex-rs/protocol/src/prompts/base_instructions/default.md:17-27`, `bef7ed0ccc` 2025-09-04 "prompt to read AGENTS.md files (#3122)"): "The scope of an AGENTS.md file is the entire directory tree rooted at the folder that contains it… More-deeply-nested AGENTS.md files take precedence… Direct system/developer/user instructions (as part of a prompt) take precedence over AGENTS.md instructions… The contents of the AGENTS.md file at the root of the repo and any directories from the CWD up to the root are included with the developer message and don't need to be re-read. When working in a subdirectory of CWD, or a directory outside the CWD, check for any AGENTS.md files that may be applicable."
- **Mid-session refresh**: global AGENTS.md re-read at every model-request boundary (even between tools in one turn); changes appended, not rewritten: "These AGENTS.md instructions replace all previously provided AGENTS.md instructions." / removal "The previously provided AGENTS.md instructions no longer apply." (`codex-rs/core/src/context/world_state/agents_md.rs:11-13`, `:44-80`) → [[codex--world-state-diff-injection]].

## Constants
| name | value | path:line |
|---|---|---|
| `project_doc_max_bytes` (shared budget) | 32768 | `codex-rs/config/src/config_toml.rs:75`, `codex-rs/config/defaults.toml:8` |
| `MAX_THREAD_INSTRUCTIONS_TOKENS` | 10_000 est. tokens (reject) | `codex-rs/core/src/agents_md_manager.rs:190` |
| `MAX_CONCURRENT_ANCESTOR_PROBES` | 256 | `codex-rs/core/src/agents_md.rs:54` |
| names per directory | `AGENTS.override.md` > `AGENTS.md` > fallbacks; first hit only | `codex-rs/core/src/agents_md.rs:43-45`, `:272-297` |
| user→project separator | `\n\n--- project-doc ---\n\n` | `codex-rs/core/src/agents_md.rs:49` |
| default root marker | `.git` | `codex-rs/core/src/agents_md.rs:10-14` |
| realtime startup context budgets | current thread 1_200 / recent work 2_200 / workspace 1_600 / notes 300 tokens; ≤40 recent threads | `codex-rs/core/src/realtime_context.rs:30-39` |

## Evolution
- 2025-05-10 `3104d81b7b` "migrate to AGENTS.md (#764)" (from codex.md / instructions.md, TS era) and `2b122da087` AGENTS.md support in Rust (with `--- project-doc ---` and `project_doc_max_bytes`).
- 2025-08-04 `063083af15` wrapped in `<user_instructions>` tags (model confused AGENTS.md with the request) → [[markdown-boundaries-ingested-inconsistently]].
- 2025-09-04 `bef7ed0ccc` scoping rules in the base prompt.
- 2025-10-15 `897d4d5f17` `AGENTS.override.md`.
- 2025-10-30 `2371d771cc` format `# AGENTS.md instructions for <dir>` + `<INSTRUCTIONS>`.
- 2025-12-22 `314937fb11` `project_root_markers`.
- 2026-06-18 `bb72e151e5` removed `child_agents_md` experiment ("adds a second model-visible explanation of hierarchical `AGENTS.md` behavior").
- 2026-06-24 `f2f80ef442` replacement/removal notices.
- 2026-07-10 `6ad0e943cc` concurrent ancestor probes (256).
- 2026-08-07 `85e0661c3b` "Cap project instructions across environments": one shared budget ("allows the total project instruction payload to grow with the number of environments" was the bug) → [[permission-context-reinjected-repeatedly]].
- 2026-08-20 `7ece061767` filesystem sandbox on AGENTS.md reads; 2026-08-21 `bd19459358` "Ignore project instructions for untrusted projects (#39837)" → [[untrusted-repo-loads-executable-config]].
- 2026-09-10 `935ac7710d` re-read global AGENTS.md at every model-request boundary → [[stale-context-files-mid-session]].
- 2026-09-11 `fc948f8c47` thread-instructions 10k-token cap (reject).
- 2026-09-16 `50d77959bf` reject path-syntax fallback names → [[context-file-discovery-filesystem-edge-cases]].

## Quirks
- Prompt says AGENTS.md is "included with the developer message" but the fragment is role `user` (stale wording).
- `docs/agents_md.md` is now a 3-line pointer to external docs.
- Silent truncation: the model never learns that a crossing file was cut ([[no-per-file-context-labels]]).

## Versus pi
- [[pi--context-file-hierarchy]]: system-prompt section, per-file `<project_instructions path>`, walk to filesystem root, not trust-gated, no budget. codex: user-role fragment, single separator, root-marker-bounded, trust-gated, 32 KiB shared budget, per-request refresh.
