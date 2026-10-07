---
type: failure
concepts: [project-trust-gate, runtime-plugin-loading]
harnesses: [pi, opencode]
---
**Symptom** — Starting pi in a freshly cloned repository silently loaded and executed the repo's `.pi/extensions`, applied `.pi/settings.json`, installed project packages and loaded `.pi/SYSTEM.md` prompt overrides — repo-controlled code ran with the user's privileges before the user did anything.

**Root cause** — Project-local config was treated like user config: discovery loaded `<cwd>/.pi/extensions` first in order (`packages/coding-agent/src/core/extensions/loader.ts:828-876`) with no notion of who authored the directory.

**Fix · [[pi]]** — `89a92207f` 2026-06-05 (PR #5332, v0.79.0) project trust gating; `718215bd9` 2026-06-08 `project_trust` extension event; `5cb4f597f` 2026-06-09 `defaultProjectTrust` + parent-folder decisions and **ungated AGENTS.md/CLAUDE.md** ("Project trust is only an input-loading guard"). Mechanism: bootstrap pass loads only global + CLI extensions with project settings forced untrusted (`packages/coding-agent/src/core/resource-loader.ts:501-521`), trust resolved (`packages/coding-agent/src/core/project-trust.ts:46-96`), untrusted ⇒ empty project settings (`settings-manager.ts:605-627`). `8562bcf66` 2026-09-29 added `.pi/mcp.json` to the gated list.

**Fix · [[opencode]]** — none, by policy. Project `opencode.json(c)`, `.opencode/plugin(s)/*.ts` and `.opencode/tool(s)/*.ts` are imported and executed on open (`packages/opencode/src/config/plugin.ts:18-25`; `packages/opencode/src/tool/registry.ts:183-197`); the only lever is `OPENCODE_DISABLE_PROJECT_CONFIG` (`packages/opencode/src/config/config.ts:420`). `SECURITY.md:33` declares "Malicious config files" out of scope: "Users control their own config; modifying it is not an attack vector".

**Lesson** — Executable or behavior-changing config supplied by a working directory must be gated by an explicit, remembered trust decision; instruction text can't be made safe that way, so don't pretend.

Related: [[project-trust-gate]] · [[runtime-plugin-loading]] · [[context-file-hierarchy]] · [[pi--project-trust-gate|pi]] · [[no-prompt-injection-defense]] · [[opencode--runtime-plugin-loading|opencode]]
