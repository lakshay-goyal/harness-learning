---
type: concept
stage: architecture
tier: variant
aliases: [codex-features, FeatureSpec, FEATURES, "Stage::UnderDevelopment", "Stage::Experimental", "[features]", "codex features", experimentalFeature/list, /experimental, feature_requirements]
harnesses: [codex]
---
A central typed registry of feature flags where each flag carries a lifecycle stage (under development → experimental → stable → deprecated → removed) that decides visibility (menu, docs), default, and whether the config key still means anything — removed keys stay parseable as no-ops.

## Why
- A fast-moving harness ships many half-built subsystems; without stages, users find and depend on unfinished ones, or old configs break when a feature is deleted.
- Removed-but-parseable keys keep old config files loading ([[session-migration]]) and double as a record of abandoned designs ([[Absences]]).
- Admins need to pin flags org-wide ([[layered-settings]] requirements).

## Design space
- **Ad-hoc booleans in settings** (pi) vs **central registry with stages** (codex).
- **Stage semantics**: hidden (under development) / user-visible opt-in menu (experimental) / kept for ad-hoc toggling (stable) / deprecated / removed no-op.
- **Removal policy**: delete key (old configs error) vs keep as no-op (codex, 41 removed keys).
- **Overrides**: user `[features]` table + CLI vs managed pins (codex `feature_requirements`) vs per-role narrowing (codex roles may only disable a small set).
- **Feature-specific config** beside the flag (codex `feature_configs.rs`).

## Implementations
- [[codex--feature-flag-stages|codex]] — `codex-rs/features`: `Stage::{UnderDevelopment, Experimental{name, menu_description, announcement}, Stable, Deprecated, Removed}`, `FEATURES` registry; census 67/3/52/4/41.

## Failures
- (none mined specific to the mechanism)

## Related
[[layered-settings]] · [[session-migration]] · [[replaceable-builtin-extension]] · [[agent-profiles]] · [[Absences]]
