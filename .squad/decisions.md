# Squad Decisions

## Active Decisions

### 2026-04-21: Branching & commit policy
**By:** Michael Jolley
**What:** Never commit against main branch. Use worktrees. Branches must follow `dev/mjolley/{title}` pattern when pushing and generating PRs.

### 2026-04-21: Scope boundary — CommandPalette.slnf only
**By:** Michael Jolley
**What:** NEVER touch files/projects not included in `src/modules/cmdpal/CommandPalette.slnf`. All work scoped exclusively to that solution filter.

### 2026-04-21: StyleCop & existing patterns
**By:** Michael Jolley
**What:** Always review StyleCop, .editorconfig, and existing code conventions. New code must maintain the same patterns that exist today. No style drift.

## Governance

- All meaningful changes require team consensus
- Document architectural decisions here
- Keep history focused on work, decisions focused on direction
