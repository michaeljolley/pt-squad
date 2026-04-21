### 2026-04-21T01:52:12Z: Branching & commit policy
**By:** Michael Jolley (via Copilot)
**What:** Never commit against main branch. Use worktrees and branches must follow the `dev/mjolley/{title}` pattern when pushing and generating PRs.
**Why:** User request — captured for team memory

### 2026-04-21T01:52:12Z: Scope boundary — CommandPalette.slnf only
**By:** Michael Jolley (via Copilot)
**What:** NEVER touch files/projects not included in the `src/modules/cmdpal/CommandPalette.slnf` solution filter. All work is scoped exclusively to that solution filter.
**Why:** User request — captured for team memory

### 2026-04-21T01:52:12Z: StyleCop & existing patterns
**By:** Michael Jolley (via Copilot)
**What:** Always review patterns for StyleCop, .editorconfig, and existing code conventions to ensure new code maintains the same patterns that exist today. No style drift.
**Why:** User request — captured for team memory
