---
type: concept
stage: architecture
tier: candidate
aliases: ["SettingsManager", "settings.json", "~/.pi/agent/settings.json", ".pi/settings.json", "layered-settings-merge", "keybindings.json", "PI_CODING_AGENT_DIR"]
harnesses: [pi]
---
Configuration resolved as global → project → CLI layers with deep merge, special rules for list-valued keys, trust gating of project layers, and concurrency-safe persistence.

## Why
- Per-repo behavior (tools, models, packages) without editing global config.
- Project config is attacker-controlled when cloning untrusted repos — project layers must be gated ([[project-trust-gate]]).
- Several harness instances share one config dir; naive read-modify-write clobbers concurrent edits ([[concurrent-settings-writes-clobber]]).

## Design space
- **Merge**: replace vs deep-merge objects (pi) with arrays replace (pi default) vs combine (pi: resource lists combine; `defaultTools` `+x/-x` modifiers append).
- **Layers**: global, project, CLI overrides (pi `applyOverrides`); env-var overrides for selected keys (e.g. `PI_TELEMETRY`, `PI_CODING_AGENT_DIR`).
- **Trust**: project layer always applied vs empty until trusted, writes throw when untrusted (pi); global-only keys (`defaultProjectTrust`, `cacheWarming`, `compaction.enabled`).
- **Persistence**: rewrite whole file vs lock + write only fields modified this session merged onto on-disk content (pi).
- **Companion files**: keybindings, models, auth, mcp, SYSTEM.md/APPEND_SYSTEM.md in the same agent dir (pi).

## Implementations
- [[pi--layered-settings|pi]] — `SettingsManager`: global + project deep merge, CLI overrides, trust-gated project layer, `proper-lockfile` with modified-field writes; namespaced keybindings file.

## Failures
- [[concurrent-settings-writes-clobber]]

## Related
[[project-trust-gate]] · [[harness-package-distribution]] · [[runtime-plugin-loading]] · [[system-prompt-override]] · [[minimal-default-toolset]] · [[differential-tui-rendering]]
