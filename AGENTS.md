# Agent Guidelines & Repository Operating System

These guidelines apply to the `app-factory` repository and its projects,
including nested Git submodules. Project-specific instructions may add local
requirements. Resolve the paths below from the `app-factory` repository root.

## Budget-First Execution

- Inspect each failing live UI state once and reuse the evidence. Do not repeat
  equivalent diagnostic popups or inspections without a new, specific hypothesis.
- Run focused regression tests during debugging, then one final relevant suite.
- Limit failed live attempts to two. After that, report the exact blocker and
  stop instead of repeating diagnostics; resume only when the user asks or
  supplies information that resolves the blocker.
- Never publish diagnostic content or retry an uncertain submission. Treat
  explicitly supplied, approved content as the intended real action, not a
  reason to repeatedly prepare and discard drafts.
- Ask for approval once per meaningful scope change, not per implementation
  step. Avoid unnecessary design gates, review rounds, and progress questions.

These user-directed rules take precedence over conflicting skill workflows.
