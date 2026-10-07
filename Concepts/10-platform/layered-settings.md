---
type: concept
stage: architecture
tier: candidate
aliases: ["SettingsManager", "settings.json", "~/.pi/agent/settings.json", ".pi/settings.json", "layered-settings-merge", "keybindings.json", "PI_CODING_AGENT_DIR", opencode.json, opencode.jsonc, OPENCODE_CONFIG, ".well-known/opencode", ai.opencode.managed, OPENCODE_DISABLE_PROJECT_CONFIG]
harnesses: [pi, opencode]
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
- **Org/enterprise layers**: remote `.well-known/opencode`, console org config, managed system dir and a macOS MDM plist that overrides everything (opencode).
- **Text substitution** `{env:VAR}`/`{file:path}` before parsing (opencode); never write the substituted form back ([[resolved-secrets-written-back-to-config]]).
- **Precedence for safety policy**: repository beats global (opencode legacy layering) vs user-global beats repository, plugins cannot add statements (opencode v2 provider policy).
- **Order-sensitive values**: permission rules as ordered arrays, not objects ([[permission-rule-order-lost-in-object-config]]).

## Implementations
- [[pi--layered-settings|pi]] — `SettingsManager`: global + project deep merge, CLI overrides, trust-gated project layer, `proper-lockfile` with modified-field writes; namespaced keybindings file.
- [[opencode--layered-settings|opencode]] — 10 layers from `.well-known` remote config through global, project, `.opencode`, env content, org, managed dir, MDM plist, `OPENCODE_PERMISSION`; deep merge, no trust gate.

## Failures
- [[concurrent-settings-writes-clobber]]
- [[permission-rule-order-lost-in-object-config]]
- [[resolved-secrets-written-back-to-config]]
- [[untrusted-repo-loads-executable-config]]
- [[flag-doc-drift]]

## Related
[[project-trust-gate]] · [[harness-package-distribution]] · [[runtime-plugin-loading]] · [[system-prompt-override]] · [[minimal-default-toolset]] · [[differential-tui-rendering]] · [[location-scoped-runtime]] · [[agent-profiles]]
