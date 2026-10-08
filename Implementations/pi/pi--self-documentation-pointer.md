---
type: implementation
harness: pi
concept: self-documentation-pointer
commit: b30a6dd77
files: [packages/coding-agent/src/core/system-prompt.ts:162-169, packages/coding-agent/src/config.ts:444-456, packages/coding-agent/src/core/tools/read.ts:96, packages/evals/src/harness.ts:257-301]
---
[[self-documentation-pointer]] in [[pi]].

## Mechanism
- `<docs>` section (default prompt only; dropped when SYSTEM.md replaces the base) (`packages/coding-agent/src/core/system-prompt.ts:162-169`):
  - Scope gate: "Pi documentation (read only when the user asks about pi itself, its SDK, extensions, themes, skills, or TUI):"
  - Absolute install paths: README, `docs/`, `examples/` from `getReadmePath/getDocsPath/getExamplesPath` = package dir (`packages/coding-agent/src/config.ts:444-456`) — not repo-relative, so they work from any cwd.
  - Resolution rule: "When reading pi docs or examples, resolve docs/... under Additional docs and examples/... under Examples, not the current working directory".
  - Topic → file routing map: extensions, themes, skills, prompt templates, TUI, keybindings, SDK, custom providers, models, packages, environment variables, MCP servers, codemode (13 topics).
  - Read-completely rules: "When working on pi topics, read the docs and examples, and follow .md cross-references before implementing"; "Always read pi .md files completely and follow links to related docs (e.g., tui.md for TUI API details)".
- Pairs with the read tool's paging contract "When you need the full file, continue with offset until complete." (`read.ts:96`) — docs exceed the 2000-line/50KB read window → [[tool-output-truncation]], [[tool-description-design]].
- Used as an **offload target**: codemode's full reference moved from tool description to `docs/codemode.md` ("Read … first", `extensions/codemode/tool.ts:154`; `6f1072cc0` — GPT-5.6 request ~5,300 → ~3,300 tok) → [[minimal-system-prompt]].
- Size share: ~45% of HEAD default prompt text is `<docs>` (findings 04 reconstruction).
- Measured: `packages/evals` documentation-lift evals run `with_docs` vs `without_docs` Docker arms; control arm strips `<docs>` via hidden `before_agent_start` extension and removes docs from disk; `verifySystemPrompt` fails closed if `<docs>` presence ≠ variant (`packages/evals/src/harness.ts:257-270, 289-301`; `d7296c063` #9635) → [[harness-evals]]. Lift numbers not published (unverified).

## Constants
| name | value | path:line |
|---|---|---|
| doc topics in map | 13 | `system-prompt.ts:167` |
| docs share of prompt | ~45% | findings 04 (reconstruction) |

## Evolution
- `0c5cbd006` 2025-11-16: "Your own documentation (including custom model setup) is at: ${readmePath}" / "Read it when users ask about features, configuration, or setup, and especially if the user asks you to add a custom model or provider."
- `7c553acd1` 2025-12-10: "Additional documentation (hooks, themes, RPC, etc.) is in: ${docsPath}" + "…or write a hook."
- `5e5bdadbf` 2025-12-17: topic map "When asked about: custom models/providers (README sufficient), themes (docs/theme.md), skills…, hooks…, custom tools…, RPC (docs/rpc.md)" (`docs/theme.md` historical: merged into `docs/themes.md` in `d79eb99cd`).
- `d1465fa0c`, `57dc16d9b`, `84b663276`, `dbdb99c48`, `67a1b9581` 2025-12-31: examples path "(hooks, custom tools, SDK)"; "When asked to create hooks, custom tools, themes, or skills: read the relevant docs AND examples, follow all .md cross-references" → per-topic "When asked to create: hooks (docs/hooks.md, examples/hooks/)…" + "Always read the doc, examples, AND follow .md cross-references before implementing" (`docs/hooks.md` historical: added `7c553acd1`, removed `c6fc08453` when hooks/custom-tools merged into extensions).
- `59d8b7948` 2026-01-06: "TUI components (docs/tui.md - has copy-paste patterns)".
- `6484ae279` 2026-01-16: Codex static instructions used `pi-internal://README.md` pseudo-URLs; reverted `4068bc556` 2026-01-17 with scope "(only when the user asks about pi itself, its SDK, extensions, themes, skills, or TUI)".
- `b846a4bfc` 2026-01-20: "only when" → "read only when".
- `89636cfe6` 2026-01-24: read description "continue with offset until complete".
- `d79eb99cd`, `d2de6d083` 2026-01-25/26: map expanded (prompt templates, keybindings, SDK, custom providers, models, packages) + two read-completely rules.
- `48b6510c1` 2026-05-19 (#4752): explicit docs/ vs examples/ resolution rule.
- `bb3d7d399` 2026-07-22: environment-variables.md added to map.
- `8562bcf66` 2026-09-29: "MCP servers (docs/mcp.md)".
- `6f1072cc0` 2026-10-01: "codemode scripts and non-LLM models such as classifiers and image models (docs/codemode.md)".
- `d7296c063` 2026-09-15: docs-lift eval isolates the effect; `1247476e6` updated eval markers to XML `<docs>`.

## Evidence commits
`0c5cbd006`, `7c553acd1`, `5e5bdadbf`, `d1465fa0c`, `57dc16d9b`, `84b663276`, `dbdb99c48`, `67a1b9581`, `59d8b7948`, `6484ae279`, `4068bc556`, `b846a4bfc`, `89636cfe6`, `d79eb99cd`, `d2de6d083`, `48b6510c1`, `bb3d7d399`, `8562bcf66`, `6f1072cc0`, `d7296c063`, `1247476e6`.

## Quirks
- Docs are the main source of prompt growth (~4× since v0) yet are gated "only when the user asks about pi itself" — cost paid every request for a minority of tasks (cache-amortized).
- Install path length varies by install method (npm global vs Nix store), so prompt bytes differ per machine but are stable per install (cache-safe).
- Custom SYSTEM.md loses the pointer entirely (no separate section).

## Failures
- [[instruction-relative-paths-resolved-from-cwd]]
- [[partial-file-read-acted-on]]
