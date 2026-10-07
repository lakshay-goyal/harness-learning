---
type: absence
harnesses: [codex]
---
# no-executable-plugins

Plugins carry skills, MCP servers, app connectors, hooks and UI metadata only — no in-process plugin code; built-in extensions are compiled Rust crates.

**What's missing**
- Manifest `paths{skills[], onboarding_skill, mcp_servers, apps, hooks}` + `interface{…}` (`codex-rs/plugin/src/manifest.rs:8-58`); first-party extensions in `codex-rs/ext/*` on a compiled API (`codex-rs/ext/extension-api/src/contributors.rs:83-430`).

**Evidence of decision**
- Design visible from the first plugin commits (`752402c4fe` 2026-03-01, `9b004e2db1` 2026-03-03) through Agent Plugins 1.0 support (`a28374e0db` 2026-07-24). Related cut: "chore: drop built-in MCPs" `32b1ae7099` 2026-05-11 ("Drop something that was never used").

**Implication**
- No third-party code inside the sandboxed harness process; admins can allow-list every capability; bundles are portable across harnesses. Cost: third parties cannot alter loop behavior except via out-of-process hooks. Contrast pi [[runtime-plugin-loading]] ([[extensibility-model]]).

Related: [[runtime-plugin-loading]] · [[plugin-tools]] · [[harness-package-distribution]] · [[codex--harness-package-distribution|codex]] · [[extensibility-model]] · [[no-auto-hot-reload]] · [[Absences]]
