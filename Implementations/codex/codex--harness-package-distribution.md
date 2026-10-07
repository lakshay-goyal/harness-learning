---
type: implementation
harness: codex
concept: harness-package-distribution
commit: 622e9e3696
files: [codex-cli/package.json:1, codex-cli/bin/codex.js:1, codex-rs/plugin/src/manifest.rs:8, codex-rs/core-plugins/src/manifest.rs:685, codex-rs/core-plugins/src/agent_plugin_manifest.rs:73, codex-rs/core-plugins/src/loader.rs:473, codex-rs/core/src/plugins/injection.rs:1]
---
[[harness-package-distribution]] in [[codex]].

Two distribution concerns: (1) shipping the harness itself (native Rust binary via npm/brew/standalone), (2) **declarative plugin bundles** (skills + MCP servers + app connectors + hooks + UI metadata — no code) from marketplaces.

## Mechanism
- `codex-rs/core-plugin-common`: "Shared plugin identities and read-only installed-package rules" (`codex-rs/core-plugin-common/src/lib.rs:1`); app declarations from plugins load via `codex-rs/ext/connectors` (`PluginAppProvider`, "Plugin app declaration loading", `codex-rs/ext/connectors/src/lib.rs:1`).
### Harness binary
- npm `@openai/codex` (`codex-cli/package.json`) ships `bin/codex.js`, a Node launcher that maps platform/arch to a target triple and per-platform optional package (`@openai/codex-linux-x64` … `@openai/codex-win32-arm64`; Linux uses musl builds) and execs the native binary, setting `CODEX_MANAGED_BY_NPM` / `CODEX_MANAGED_BY_BUN` so self-update knows the installer (`codex-cli/bin/codex.js:1-40`, `:200-236`). Install method classified Standalone/Npm/Bun/Brew (`codex-rs/install-context/src/lib.rs:55-77`); `codex update`.
- `codex-cli/` today only `bin/`, `scripts/`, `package.json` → [[no-typescript-cli]].
- Release (`rust-release.yml`): per-target binaries, sigstore bundles, cosign for Linux voice archives, macOS signing via protected `codesigning` env, Python runtime wheel; separate workflows for Windows, zsh build, rusty_v8, Python SDK/runtime, npm SDK. Cadence: 270 stable `rust-v0.x.y` tags (1,470 incl. alphas), a stable release every 2–3 days (M8 repo facts). Dual build Cargo + Bazel (`MODULE.bazel`), PR gate `bazel.yml`, full Cargo matrix post-merge (`.github/workflows/README.md`).

### Plugin bundles
- **Manifest** = name, version, description, keywords, `paths{skills[], onboarding_skill, mcp_servers, apps, hooks}`, `interface{display_name, descriptions, developer, category, capabilities, urls, default_prompt, brand_color, icons, screenshots}` (`codex-rs/plugin/src/manifest.rs:8-58`). No executable plugin API → [[no-executable-plugins]].
- **Accepted manifest locations**: `.codex-plugin/plugin.json`; alternate `.claude-plugin/plugin.json` (cross-harness compatibility, `37161bc76e` 2026-04-16, `codex-rs/core-plugins/src/manifest.rs:685`); Agent Plugins 1.0 root `plugin.json` with `$schema` check and Codex extras under an inline `com.openai` extension (`a28374e0db` 2026-07-24; `codex-rs/core-plugins/src/agent_plugin_manifest.rs:73`, `:141`; `codex-rs/utils/plugins/src/plugin_namespace.rs:15-17`).
- **Marketplaces**: repo-local `.agents/plugins/marketplace.json` (+ `api_marketplace.json`) (`codex-rs/core-plugins/src/loader.rs:473-480`), git/npm sources (`npm_source.rs`), remote catalog, sharing (`plugin/share/*`), managed marketplace policy (`marketplace_policy.rs`), startup sync, recommended plugins; CLI `codex plugin`, app-server `plugin/*`, `marketplace/*`, TUI `/plugins`.
- **Model exposure**: explicit plugin @-mention → developer hint pointing the model at the plugin's MCP servers, apps and skill prefix (`codex-rs/core/src/plugins/injection.rs` `build_plugin_injections`); plugin skills appear under a plugin prefix → [[skill-progressive-disclosure]].
- **Policy**: requirements can restrict MCP servers/plugins/marketplaces/apps ([[codex--layered-settings|layered-settings]]); reserved marketplace names protected (`0acf302db5` 2026-08-18 "Prevent marketplace identity spoofing") → [[untrusted-repo-loads-executable-config]].
- **Import from other harnesses**: Claude Code marketplace names and `.cursor-plugin/marketplace.json` → [[external-agent-import]].

## Evolution
- 2025-05-12 `73fe1381aa` `--native`: npm package starts shipping the Rust binary; 2025-08-08 `408c7ca142` TypeScript code removed.
- 2026-03-01 `752402c4fe` load from plugins; first plugin commits 2026-03-03 `9b004e2db1`, `082682a628`; extracted to crate 2026-04-15 `48cf3ed7b0`.
- 2026-04-16 `37161bc76e` alternate manifest paths; 2026-07-24 `a28374e0db` Agent Plugins 1.0 manifests.
- 2026-08-18 `0acf302db5` marketplace identity spoofing fix.

## Versus pi
- [[pi--harness-package-distribution]]: pi packages carry **code** (TS extensions) plus skills/prompts/themes from npm/git/URL/path with pinning and filters; codex plugins carry only declarative capabilities (skills, MCP, apps, hooks) and are discovered via marketplaces, with manifest compatibility for Claude Code and the Agent Plugins 1.0 format. See [[extensibility-model]].
