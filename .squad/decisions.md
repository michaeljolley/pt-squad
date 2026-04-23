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

### 2026-04-22: DI Migration — Full Investigation Complete
**By:** Hawk, Scarlett, Snake Eyes (collaborative analysis)
**Issue:** General code quality / DI modernization
**What:** Completed comprehensive analysis across three dimensions:
  - **Hawk:** Full dependency graph (30+ services, 5 levels, zero circular dependencies)
  - **Scarlett:** UI layer audit (12 files, 47 service calls; 3-phase migration strategy)
  - **Snake Eyes:** ViewModel/service analysis (2 quick wins: HotkeyManager injection, DefaultContextMenuFactory DI)
**Decision:** DI migration is **architecturally feasible** with no blockers. No circular dependencies. Clear path forward.
**Details:** See `.squad/orchestration-log/2026-04-22T02-31-00Z-{hawk,scarlett,snake-eyes}.md` for detailed findings.

### 2026-04-22: UI Layer DI Strategy — Property Injection Pattern
**By:** Scarlett (UI Dev)
**Status:** Ready for Implementation
**What:** XAML pages/controls cannot use constructor injection (parameterless constructor constraint). Solution: property injection with careful App/factory coordination.
**Migration Phases:**
  - **Phase 1 (EASY):** SearchBar, AppearancePage, ExtensionsPage, GeneralPage, InternalPage — validate property-injection pattern (~12 calls removed)
  - **Phase 2 (MEDIUM):** ListPage, DockWindow, ContextMenu, FallbackRanker, DockSettingsPage — refactor ViewModels (~17 calls removed)
  - **Phase 3 (HARD):** MainWindow, ShellPage — careful App lifecycle coordination (~15 calls removed)
**Critical Risk:** Services injected via properties MUST be set BEFORE control/page use. Requires tight App/factory coordination.
**Next:** Team approval on migration strategy, then prototype with one EASY file.

### 2026-04-22: Quick Wins — HotkeyManager & DefaultContextMenuFactory
**By:** Snake Eyes
**Status:** Ready for Immediate Implementation
**What:** Two low-risk DI improvements:
  1. **Inject HotkeyManager into TopLevelViewModel** — eliminates `_serviceProvider.GetService<HotkeyManager>()` in Hotkey property
  2. **Replace DefaultContextMenuFactory singleton with DI** — register as `IContextMenuFactory` singleton, inject instead of using static `Instance`
**Risk:** Low. Both are straightforward constructor additions with no circular dependencies.
**Impact:** Reduces service locator usage, improves testability and clarity.

### 2026-04-21: Tag overflow fix approach for #38317
**By:** Michael Jolley (via Copilot)
**What:** Two-pronged fix for Command Palette list item tag overflow:
  1. Prioritize title text — title always gets layout priority, tags get remaining space. Handle overflow with ellipsis appropriately.
  2. Cap visible tags at 3 — if more than 3 tags, show only 3 and append a `[+ N]` badge indicating how many more exist.
**Why:** User design decision for issue #38317 — ensures title is always readable while keeping tags useful.

### 2026-04-21: Build instructions for CmdPal
**By:** Michael Jolley (via Copilot)
**What:** Build CmdPal by targeting the `Microsoft.CmdPal.UI` project in `src/modules/cmdpal/CommandPalette.slnf`.
**Why:** User directive — correct build procedure for Command Palette work.

### 2026-04-22: Use Synthetic TagViewModels for Overflow Badges
**By:** Scarlett (UI Dev)
**Issue:** #47140
**What:** Replace the `Border` overflow badge with a synthetic `TagViewModel` appended to `VisibleTags`. The overflow "+N" indicator is now rendered by the same `ItemsRepeater` + `TagTemplate` pipeline as regular tags.
**Why:** Addresses niels9001's perf review feedback — eliminates one `Border` + `TextBlock` from every list item's visual tree, while removing 3 properties (`OverflowTagCount`, `HasOverflowTags`, `OverflowTagText`) and their change notifications.
**Impact:**
  - Perf: Eliminates visual tree bloat
  - Simplicity: Fewer properties, cleaner VM logic
  - Pattern: Establishes "synthetic VM sentinel" as preferred approach for conditional badges in repeated templates


### 2026-04-23: DI Migration Design Review — APPROVED WITH CONDITIONS
**By:** Hawk, Scarlett, Snake Eyes, Flint (unanimous)
**Ceremony:** Design Review — 4-agent sign-off
**Status:** ✅ APPROVED WITH CONDITIONS  
**What:** Design review of DI migration plan Phases 0–3 with architecture, UI feasibility, ViewModel/service layer, and testing strategy validation.

**Key Decisions:**
1. **Dynamic extension providers do NOT use IServiceProvider** — IEnumerable<ICommandProvider> injection is safe for built-ins (no circular dependencies confirmed by Hawk)
2. **DockSettingsPage moved from Phase 1 to Phase 2** — requires ViewModel refactoring patterns; not a true quick win
3. **Prototype SettingsPageBase with AppearancePage first** — establishes pattern before scaling to other files
4. **IPageFactory must support parameterized navigation** — required for ExtensionPage parameter passing
5. **Inject both HotkeyManager AND AliasManager into TopLevelViewModel** — replaces service locator calls in Hotkey property and AliasText setter
6. **Baseline tests required BEFORE Phase 0 code changes** — 3 critical classes (TopLevelCommandManager, HotkeyManager, DefaultContextMenuFactory) have zero test coverage; ~25–30 tests needed across all phases
7. **All 4 team members approved the plan** — Hawk (architecture), Scarlett (UI feasibility), Snake Eyes (ViewModel/service layer), Flint (testing strategy)

**Implementation Conditions:**
- Phase 0: Tests 1.0–1.3 written and passing before code changes ✅ COMPLETE (Flint Phase 0a)
- Phase 1: Move DockSettingsPage to Phase 2; file list validated (5 genuine quick wins)
- Phase 2: IPageFactory and SettingsPageBase patterns proven
- Phase 3: Hawk code review required before merge; ShellPage prototype required

**Next Steps:**
- Snake Eyes: Prepare Phase 0 PR (HotkeyManager, AliasManager, DefaultContextMenuFactory, TaskScheduler)
- Scarlett: Design IPageFactory with SettingsWindow integration; prototype SettingsPageBase with AppearancePage
- Hawk: Schedule Phase 3 code review

**Risk Level:** LOW-MEDIUM | **Confidence:** HIGH | **Blockers:** None

### 2026-04-23: Phase 0a Baseline Test Coverage — COMPLETE
**By:** Flint (Tester)
**Status:** ✅ COMPLETE
**What:** Wrote 25 baseline unit tests for Phase 0 critical classes before code changes.
  - **TopLevelCommandManagerTests:** 12 tests (constructor, TaskScheduler resolution, LoadBuiltinsAsync 0/1/2 providers, Dispose)
  - **HotkeyManagerTests:** 7 tests (constructor, UpdateHotkey add/remove/replace/conflict)
  - **DefaultContextMenuFactoryTests:** 6 tests (singleton pattern, IContextMenuFactory, null/empty items, no-op)
**Key Finding:** The `!` null-forgiving operator on `GetService<TaskScheduler>()!` is compile-time only. At runtime, constructor silently stores null if TaskScheduler is unregistered. Phase 0b must ensure TaskScheduler is always registered.
**Impact:** Safety net established for Phase 0b code changes. Ready for Snake Eyes to proceed with HotkeyManager/AliasManager injection, DefaultContextMenuFactory DI conversion, TaskScheduler parameter injection.
**Files:** TopLevelCommandManagerTests.cs, HotkeyManagerTests.cs, DefaultContextMenuFactoryTests.cs

## Governance

- All meaningful changes require team consensus
- Document architectural decisions here
- Keep history focused on work, decisions focused on direction

