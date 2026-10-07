---
type: failure
concepts: [protected-workspace-metadata, os-level-sandbox, project-trust-gate]
harnesses: [codex]
---
**Symptom** — With a workspace-write sandbox the agent could write files inside the workspace that the harness or the user's tooling later *executes or trusts* with more authority than the sandbox: `.git/hooks` (run on the user's next commit), `.codex/` (project config + `rules/*.rules` that auto-approve commands), `.agents/` (skills), `.aws/` (profiles selecting credential-helper executables) — or simply `mkdir .codex` the first time.

**Root cause** — "Writable workspace" was defined as "everything under cwd"; the sandbox enforced *now*, but the written files became authority *later*, outside any approval.

**Fix · [[codex]]** — carve-outs added one vector at a time: 2025-08-01 `80555d4ff2` `.git` read-only on Seatbelt; 2025-12-15 `bef36f4ae7` `.codex`; 2026-02-03 `9a487f9c18` `.agents` ("make $PWD/.agents read-only like $PWD/.codex"); 2026-03-26 `86764af684` first-time `.codex` creation on Linux + macOS; 2026-04-28/29 `0156b1e61f` / `0670d8971a` / `74f06dcdfb` generic protected metadata names across backends; 2026-09-25 `1d804e91b7` `.aws` ("AWS profiles can select credential helpers that the application executes"). Now `PROTECTED_METADATA_PATH_NAMES` (`codex-rs/protocol/src/permissions.rs:40-51`), per-name rules (`codex-rs/protocol/src/permissions.rs:2346-2390`), Seatbelt `require-not literal + subpath` (`codex-rs/sandboxing/src/seatbelt.rs:565-604`).

**Lesson** — A writable sandbox root must exclude every path whose contents will later run or be trusted with more authority than the agent; the list grows with every new tool convention (hooks, agent config, skills, credential helpers).

Related: [[protected-workspace-metadata]] · [[os-level-sandbox]] · [[project-trust-gate]] · [[codex--protected-workspace-metadata|codex]] · [[sandbox-path-binding-races]] · [[untrusted-repo-loads-executable-config]]
