---
type: failure
concepts: [tool-description-design, deferred-tool-loading]
harnesses: [codex]
---
**Symptom** — `tool_suggest` was called when the model actually needed `tool_search` (to load an already-installed tool); the model used `list_mcp_resources` for tool discovery; it reasoned in "connector" terms the descriptions didn't use.

**Root cause** — Two discovery-ish tools with overlapping names ("suggest" vs "search"), and descriptions in harness vocabulary rather than the model's.

**Fix · [[codex]]**
- `bc48b9289a` 2026-03-12 "Update tool search prompts (#14500)" — "Add mentions of connectors because model always think in connector terms in its CoT"; "always use `tool_search` instead of `list_mcp_resources` or `list_mcp_resource_templates` for tool discovery" (`codex-rs/core/src/tools/handlers/tool_search_spec.rs:94`).
- `8ce48f9968` 2026-04-29 tightened triggers ("Use this ONLY when all of the following are true…", "IMPORTANT: DO NOT call this tool in parallel with other tools.").
- `f88701f5c8` 2026-05-02 renamed `tool_suggest` → `request_plugin_install`: "Tool suggest still misfires when model needs tool_search… rephrase "suggestion" to "install"… disambiguate "the tool" vs "the plugin/connector"" (`codex-rs/core/templates/search_tool/request_plugin_install_description.md:1-29`).

**Lesson** — The tool name is the strongest part of its description: name tools after their side effect and use the vocabulary the model already thinks in.

Related: [[tool-description-design]] · [[deferred-tool-loading]] · [[codex--tool-description-design|codex]] · [[codex--deferred-tool-loading|codex discovery]]
