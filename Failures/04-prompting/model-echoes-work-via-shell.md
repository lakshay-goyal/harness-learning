---
type: failure
concepts: [minimal-system-prompt, guideline-softening]
harnesses: [pi]
---
**Symptom** — When summarizing what it had done, the model ran `cat`/heredoc/`echo` through bash to "display" its summary instead of answering in plain text (early models, Nov 2025).

**Root cause** — Habit from shell-centric agent training; nothing in the v0 prompt said the final answer is plain assistant text.

**Fix · [[pi]]**
- `b3d4478b6` 2025-11-20 (Release v0.7.23): + "When summarizing your actions, output plain text directly - do NOT use cat or bash to display what you did" (CHANGELOG: "instruct agent to output plain text summaries directly instead of using cat or bash commands to display what it did").
- `235b247f1` 2026-03-22: rule **silently dropped** while moving tool guidance into `ToolDefinition`s (CHANGELOG `:2914` only "Cleaned up `buildSystemPrompt()`"); not re-added in 6+ months (unverified that the symptom vanished).

**Lesson** — Model-habit rules are temporary; record why they were added so you can retire them deliberately when models outgrow the habit.

Related: [[minimal-system-prompt]] · [[guideline-softening]] · [[pi--minimal-system-prompt|pi]]
