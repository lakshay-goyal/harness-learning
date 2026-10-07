---
type: implementation
harness: pi
concept: project-trust-gate
commit: b30a6dd77
files: [packages/coding-agent/src/core/trust-manager.ts:30, packages/coding-agent/src/core/trust-manager.ts:186, packages/coding-agent/src/core/project-trust.ts:46, packages/coding-agent/src/core/resource-loader.ts:501, packages/coding-agent/src/core/settings-manager.ts:605, packages/coding-agent/docs/security.md:28, packages/coding-agent/src/extensions/mcp/config.ts:133]
---
[[project-trust-gate]] in [[pi]].

## Mechanism
- **Trigger** `hasTrustRequiringProjectResources(cwd)` (`packages/coding-agent/src/core/trust-manager.ts:186-208`): true iff `<cwd>/.pi/` contains any of `settings.json, mcp.json, extensions, skills, prompts, themes, SYSTEM.md, APPEND_SYSTEM.md` (`TRUST_REQUIRING_PROJECT_CONFIG_RESOURCES`, `:30-39`), or `.agents/skills` exists in cwd or **any ancestor**, excluding `~/.agents/skills` "even when cwd is $HOME" (`:180-185`). Bare `.pi` dir does not trigger (`packages/coding-agent/docs/security.md:45`). No trigger ⇒ resolved as **trusted** without asking (`packages/coding-agent/src/core/project-trust.ts:50-52`).
- **Decision order** `resolveProjectTrusted` (`project-trust.ts:46-96`):
  1. CLI `--approve`/`-a` or `--no-approve`/`-na` override (`:47-49`).
  2. `project_trust` event to **user/global + CLI `-e` extensions only**; first non-`undecided` yes/no wins; `remember:true` persists it (`:54-70`; event `packages/coding-agent/src/core/extensions/types.ts:686-709`, `packages/coding-agent/src/core/extensions/runner.ts:295-322`; `718215bd9`).
  3. Nearest saved decision walking up parents in `~/.pi/agent/trust.json` (`:72-75`).
  4. `defaultProjectTrust` `always|never|ask` (default `"ask"`, **global settings only**, `packages/coding-agent/src/core/settings-manager.ts:151`; `packages/coding-agent/docs/settings.md:34`) (`:77-84`).
  5. No UI (print/json/rpc) ⇒ **false** (`:86-88`); else interactive prompt (`:90-95`).
- **Prompt**: "Trust project folder? … This allows pi to load .pi settings and resources, install missing project packages, and execute project extensions." (`project-trust.ts:24-26`). Options: Trust / Trust parent folder (saves parent, clears child) / Trust (this session only) / Do not trust / Do not trust (this session only) (`trust-manager.ts:66-97`). `/trust` slash command saves a decision for future sessions (`packages/coding-agent/src/core/slash-commands.ts:36`).
- **Two-pass load**: bootstrap pass forces project settings untrusted and loads only global + CLI extensions (`packages/coding-agent/src/core/resource-loader.ts:501-507`), resolves trust with them (`:518-521`), then full load. Untrusted ⇒ `projectSettings = {}` and writes to project settings throw (`settings-manager.ts:605-627,685-689`).
- **Gated consumers**: `.pi/SYSTEM.md` / `APPEND_SYSTEM.md` only if `isProjectTrusted()` (`resource-loader.ts:1210-1226`); project packages (`docs/packages.md:19`); project MCP servers (`docs/mcp.md:32`); `ctx.isProjectTrusted` exposed to extensions (`extensions/types.ts:325-364`).
- **MCP credential steering blocked independently of trust**: project `mcp.json` entries may not use `"auth": {"provider": …}` — "so a repository cannot pick where the credential goes" (`packages/coding-agent/src/extensions/mcp/config.ts:25-26,133-135`).
- **Not gated**: `AGENTS.override.md`, `AGENTS.md`, `CLAUDE.md` "load regardless of project trust" (`docs/security.md:57`) — see [[context-file-hierarchy]].
- **Known hole (documented)**: project `sessionDir` is read before trust is resolved (`docs/security.md:31`).
- **Stance**: "Project trust is only an input-loading guard … Prompt injection from repository files … is expected local-agent risk and cannot be reliably prevented by pi." (`5cb4f597f` diff of `docs/security.md`); "does not limit what tool calls can access" (`docs/security.md:33`).
- **Store concurrency**: `trust.json` writes under `proper-lockfile`, 10 attempts × 20 ms synchronous busy-wait (`trust-manager.ts:138-177`).
- **Layered plugin confirmation**: subagent example adds its own "Project agents are repo-controlled. Only continue for trusted repositories." prompt for `.pi/agents`, skipped when project trusted (`examples/extensions/subagent/index.ts:484-545`; `8af7690c4`).

## Constants
| name | value | path:line |
|---|---|---|
| `defaultProjectTrust` | `"ask"` (global-only) | packages/coding-agent/src/core/settings-manager.ts:151 |
| trust-requiring entries | 8 `.pi/*` names + ancestor `.agents/skills` | packages/coding-agent/src/core/trust-manager.ts:30-39 |
| trust lock | 10 × 20 ms busy-wait | packages/coding-agent/src/core/trust-manager.ts:141-142 |
| store | `~/.pi/agent/trust.json` | packages/coding-agent/docs/security.md |

## Evolution
- 2026-06-05 `89a92207f` "add project trust gating" (PR #5332, v0.79.0; `packages/coding-agent/CHANGELOG.md:1679,1687`): gated "project-local settings, resources, **instructions**, and packages"; triggers included `AGENTS.md`/`CLAUDE.md` in cwd or ancestors. Motivation: untrusted repos could run `.pi/extensions` ([[untrusted-repo-loads-executable-config]]).
- 2026-06-08 `718215bd9` `project_trust` extension event; `ce3a72444` security model doc.
- 2026-06-09 `5cb4f597f` (Armin Ronacher) "Improved project approval settings": **ungated AGENTS.md/CLAUDE.md** (removed from triggers), added `defaultProjectTrust` and parent-directory decisions; rationale in diff: trust "prevents a repository from silently changing pi's settings or extensions"; injection via "context files" is expected risk.
- 2026-06-12 `b4bff7f0d` (#5619/PR #5674): ignore global `~/.pi/agent` state when running from `$HOME`; `pi update` uses only saved/explicit trust (`packages/coding-agent/CHANGELOG.md:1619`) — [[trust-scope-includes-agent-config-dir]].
- 2026-08-18 `8af7690c4` subagent example skips its repo-agent prompt in trusted projects (#8261).
- 2026-09-29 `8562bcf66`: `.pi/mcp.json` added to the trust list; project MCP `auth.provider` forbidden.

## Evidence commits
`89a92207f`, `718215bd9`, `ce3a72444`, `5cb4f597f`, `b4bff7f0d`, `8af7690c4`, `8562bcf66`


## Quirks
- Repos with no trust-requiring resources are silently "trusted" (`project-trust.ts:50-52`) — trust prompts only appear when there is something to gate.
- Non-trust-listed project files read by plugins are ungated: the `sandbox/` example reads `<cwd>/.pi/sandbox.json` (can set `enabled:false`) with no trust check, and that file alone does not trigger a prompt ([[repo-config-disables-sandbox-plugin]]).
- Reversal (gated → ungated instructions in 4 days) leaves pi's position: trust ≠ injection defence ([[no-prompt-injection-defense]]).

## Failures
- [[untrusted-repo-loads-executable-config]] · [[trust-scope-includes-agent-config-dir]] · [[repo-config-disables-sandbox-plugin]]
