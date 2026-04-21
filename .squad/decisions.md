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

### 2026-04-21: Branching & commit policy (confirmation)
**By:** Michael Jolley (via Copilot)
**What:** Never commit against main branch. Use worktrees and branches must follow the `dev/mjolley/{title}` pattern when pushing and generating PRs.

### 2026-04-21: Scope boundary — CommandPalette.slnf only (confirmation)
**By:** Michael Jolley (via Copilot)
**What:** NEVER touch files/projects not included in the `src/modules/cmdpal/CommandPalette.slnf` solution filter. All work is scoped exclusively to that solution filter.

### 2026-04-21: StyleCop & existing patterns (confirmation)
**By:** Michael Jolley (via Copilot)
**What:** Always review patterns for StyleCop, .editorconfig, and existing code conventions to ensure new code maintains the same patterns that exist today. No style drift.

### 2026-04-21: CmdPal pluralization fix strategy
**By:** Scarlett (UI Dev)
**Issue:** #47110
**What:** Fixed pluralization in Extensions settings by adding 4 singular-form resource strings and rewriting the `ExtensionSubtext` property with tuple pattern matching. Covers all combinations: 1 command, 1 fallback, both singular, mixed.
**Why:** User-facing text must correctly pluralize "command" vs "commands" and "fallback command" vs "fallback commands" based on actual counts.

### 2026-04-21: Comprehensive test strategy — pluralization
**By:** Flint (Tester)
**Issue:** #47110
**What:** Test suite covers 17 scenarios: zero, one, two, five, and 100 command/fallback combinations, plus edge cases (disabled providers, built-in providers). Validates both singular and plural forms across all permutations.
**Why:** Pluralization logic is error-prone; comprehensive coverage ensures regressions are caught early.

## Governance

- All meaningful changes require team consensus
- Document architectural decisions here
- Keep history focused on work, decisions focused on direction
