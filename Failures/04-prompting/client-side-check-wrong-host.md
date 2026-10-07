---
type: failure
concepts: [prompt-template-expansion, remote-host-trust]
harnesses: [codex]
---
**Symptom** — `/init`'s "AGENTS.md already exists?" guard ran on the TUI machine's filesystem, which is wrong when the TUI drives a remote app-server (`--remote`): it could skip `/init` because of a local file or allow overwriting a remote one.

**Root cause** — A filesystem precondition evaluated in the client process while tools execute on another host.

**Fix · [[codex]]** — `8e69d29521` 2026-06-09 "Reduce TUI legacy core dependencies (#26711)": deleted the TUI `init_target.exists()` guard ("AGENTS.md already exists here. Skipping /init to avoid overwriting it.") — "checking the TUI's local filesystem for `/init` is incorrect" — and added to the prompt "Before writing, check whether AGENTS.md already exists in the current working directory. If it does, do not overwrite or modify it." (`codex-rs/tui/assets/prompt_for_init_command.md:2`); test `slash_init_skips_when_project_doc_exists` replaced by `slash_init_does_not_depend_on_loaded_instruction_sources`.

**Lesson** — Once UI host and execution host can differ, filesystem preconditions must be evaluated where tools run — sometimes simplest by delegating them to the model's tools.

Related: [[prompt-template-expansion]] · [[remote-host-trust]] · [[client-server-session-split]] · [[codex--prompt-template-expansion|codex]]
