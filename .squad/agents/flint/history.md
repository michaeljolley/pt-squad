# Project Context

- **Owner:** Michael Jolley
- **Project:** Microsoft Command Palette (CmdPal) — a PowerToys utility providing a command launcher with extensible plugin architecture
- **Stack:** C# (WinUI 3 / WinAppSDK), C++/WinRT, XAML, WinRT IDL
- **Scope:** src/modules/cmdpal/ and CommandPalette.slnf only
- **Created:** 2026-04-20

## Core Context

This section contains historical learnings, architecture reviews, and analysis from prior sessions summarized for reference. See dated sections below for current session notes.

**Architecture Summary:** CmdPal is a layered architecture with UI (WinUI 3) → ViewModels (MVVM Toolkit) → Common/Services → Extension SDK (WinRT IDL) → 17 Built-in Extensions. Clean dependency graph with no circular dependencies. DI via Microsoft.Extensions.DependencyInjection. Messaging via MVVM Community Toolkit. Extension model uses ICommandProvider interface; extensions loaded dynamically as in-proc COM objects. Full test coverage across 16 test projects.

**Prior Session Findings (2026-04-20 to 2026-04-22):** Comprehensive analysis completed by team:
- **Service Locator Analysis:** Identified 47 anti-pattern calls across 12 UI files; 30+ services in dependency graph (5 levels deep, zero circular dependencies)
- **Quick Wins Identified:** HotkeyManager injection, DefaultContextMenuFactory DI conversion, TaskScheduler parameter injection
- **UI Layer Strategy:** XAML pages use property injection (parameterless constructor constraint); 3-phase migration identified (Easy/Medium/Hard files)
- **Testing Status:** 16 existing test projects; 3 critical services need test coverage baseline before implementation

---

## 2026-04-23: DI Migration Design Review Ceremony

**Date:** 2026-04-23  
**Verdict:** ✅ APPROVED WITH CONDITIONS (4/4 agents)  
**Ceremony:** Multi-agent design review - Architecture, UI, ViewModel/Services, QA  

**Key Decisions from Review:**
1. Dynamic extension providers do NOT use IServiceProvider — IEnumerable<ICommandProvider> injection is safe
2. DockSettingsPage moved from Phase 1 to Phase 2  
3. Prototype SettingsPageBase with AppearancePage first  
4. IPageFactory must support parameterized navigation  
5. Inject both HotkeyManager AND AliasManager into TopLevelViewModel  
6. Baseline tests required BEFORE Phase 0 code changes (~25–30 tests across all phases)  
7. All 4 team members approved the plan

**Implementation Path:**
- Phase 0: Tests 1.0–1.3, then HotkeyManager, AliasManager, DefaultContextMenuFactory, TaskScheduler injection
- Phase 1: 5 genuine quick wins (SearchBar, AppearancePage, ExtensionsPage, GeneralPage, InternalPage)
- Phase 2: IPageFactory & SettingsPageBase proven; medium complexity files  
- Phase 3: Hawk review + ShellPage prototype required

**Next:** Snake Eyes (Phase 0 PR), Flint (tests), Scarlett (IPageFactory), Hawk (Phase 3 review)

---

## Learnings

### Test Infrastructure (ViewModels)
- **Test project:** `src/modules/cmdpal/Tests/Microsoft.CmdPal.UI.ViewModels.UnitTests/`
- **Build:** `tools\build\build.cmd` from the test project dir. Must nuke `obj\x64` to defeat incremental no-op on new files.
- **Run:** `vstest.console.exe` against `<project>\x64\Debug\WinUI3Apps\CmdPal\tests\*.dll` (not the solution-level `x64\Debug` path — that may be stale).
- **Patterns:** `[TestClass]` + `[TestMethod]`, `partial class` required when inner types extend WinRT base classes (e.g., `CommandProvider`). Existing tests use `TaskScheduler.Default`, `CommandProviderContext.Empty`, `DefaultContextMenuFactory.Instance` as lightweight stubs.
- **Moq:** Available via `<PackageReference Include="Moq" />`. Used for `ISettingsService`, `ICommandProviderCache`. Capture lambda transforms via `Callback<Func<SettingsModel, SettingsModel>, bool>(...)` pattern.
- **DI in tests:** Use `ServiceCollection` + `BuildServiceProvider()` to create real `IServiceProvider` with registered singletons. This mirrors production DI wiring.
- **WinRT base classes:** `CommandProvider` from `Microsoft.CommandPalette.Extensions.Toolkit` can be extended in tests as `TestCommandProvider` with empty `TopLevelCommands()`.
- **Null-forgiving `!`:** The `!` operator on `GetService<TaskScheduler>()!` is compile-time only. At runtime, the constructor silently stores null — no `NullReferenceException` thrown.
- **`LoadBuiltinsAsync` dependencies:** Needs both `TaskScheduler` and `ISettingsService` registered in the service provider. The `CommandProviderWrapper` constructor reads `Id`, `DisplayName`, `Icon`, `Settings`, hooks `ItemsChanged`, and calls `InitializeWithHost`.

### Files Created
- `TopLevelCommandManagerTests.cs` — 12 tests: constructor injection, initial state, dispose, LoadBuiltinsAsync with 0/1/2 providers
- `HotkeyManagerTests.cs` — 7 tests: constructor, UpdateHotkey with add/remove/replace/conflict
- `DefaultContextMenuFactoryTests.cs` — 6 tests: singleton instance, IContextMenuFactory, null/empty items, no-op method

