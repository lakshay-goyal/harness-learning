---
type: concept
stage: architecture
tier: must-have
aliases: ["SettingsManager", "settings.json", "~/.pi/agent/settings.json", ".pi/settings.json", "layered-settings-merge", "keybindings.json", "PI_CODING_AGENT_DIR", opencode.json, opencode.jsonc, OPENCODE_CONFIG, ".well-known/opencode", ai.opencode.managed, OPENCODE_DISABLE_PROJECT_CONFIG, ConfigLayerStack, ConfigLayerSource, config.toml, requirements.toml, managed_config.toml, "Constrained<T>", configRequirements/read, cloud-config, "--strict-config"]
harnesses: [pi, opencode, codex]
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
- **Separate constraint stack** (allow-lists, exact pins, feature pins, managed-hooks-only) distinct from values ✔ codex; values wrapped in `Constrained<T>`, writes overlapping an exact requirement rejected.
- **Enterprise sources**: MDM, `/etc` system file, HMAC-signed cloud bundle with 15 min refresh + 1 h cache ✔ codex.
- **Profiles as separate files** (`<name>.config.toml`) ✔ codex.
- **Provenance**: per-key origin + per-layer fingerprint for optimistic-concurrency writes ✔ codex (vs pi lockfile + modified-field merge).
- **Strict mode**: reject unknown keys (`--strict-config`) with generated JSON schema ✔ codex.
- **Trust (codex)**: untrusted project layer loaded but disabled with a reason (config, hooks, exec policies).
- **Org/enterprise layers**: remote `.well-known/opencode`, console org config, managed system dir and a macOS MDM plist that overrides everything (opencode).
- **Text substitution** `{env:VAR}`/`{file:path}` before parsing (opencode); never write the substituted form back ([[resolved-secrets-written-back-to-config]]).
- **Precedence for safety policy**: repository beats global (opencode legacy layering) vs user-global beats repository, plugins cannot add statements (opencode v2 provider policy).
- **Order-sensitive values**: permission rules as ordered arrays, not objects ([[permission-rule-order-lost-in-object-config]]).

## Implementations
- [[pi--layered-settings|pi]] — `SettingsManager`: global + project deep merge, CLI overrides, trust-gated project layer, `proper-lockfile` with modified-field writes; namespaced keybindings file.
- [[codex--layered-settings|codex]] — 10 provenance-tagged TOML layers (packaged → MDM → system → cloud bundle → user → profile → project → `-c` → legacy managed) with per-key origins and fingerprints, plus a separate requirements constraint stack enforced via `Constrained<T>`.
- [[opencode--layered-settings|opencode]] — 10 layers from `.well-known` remote config through global, project, `.opencode`, env content, org, managed dir, MDM plist, `OPENCODE_PERMISSION`; deep merge, no trust gate.

## Failures
- [[concurrent-settings-writes-clobber]]
- [[untrusted-repo-loads-executable-config]]
- [[session-config-not-restored-on-resume]] (08-state) — --resume reset thinking level to off (#342); resuming appended a spurious thinking_level_change entry…
- [[permission-rule-order-lost-in-object-config]]
- [[resolved-secrets-written-back-to-config]]
- [[flag-doc-drift]]

## Related
[[project-trust-gate]] · [[harness-package-distribution]] · [[runtime-plugin-loading]] · [[system-prompt-override]] · [[minimal-default-toolset]] · [[differential-tui-rendering]] · [[feature-flag-stages]] · [[location-scoped-runtime]] · [[agent-profiles]]
