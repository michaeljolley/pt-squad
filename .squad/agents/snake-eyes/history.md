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

## 2026-04-23: Phase 0a Baseline Tests Complete — Ready for Phase 0b

**Date:** 2026-04-23
**Agent:** Flint (Tester)
**Status:** ✅ COMPLETE

**What:** Wrote 25 baseline unit tests for 3 critical classes before DI migration code changes.
- **TopLevelCommandManagerTests.cs** (12 tests): Constructor injection, TaskScheduler resolution, LoadBuiltinsAsync with 0/1/2 providers, Dispose
- **HotkeyManagerTests.cs** (7 tests): Constructor, UpdateHotkey add/remove/replace/conflict scenarios
- **DefaultContextMenuFactoryTests.cs** (6 tests): Singleton pattern, IContextMenuFactory implementation, null/empty handling

**Key Finding:** The `!` null-forgiving operator on `GetService<TaskScheduler>()!` is compile-time only. At runtime, the constructor does NOT throw if TaskScheduler is unregistered — it silently stores null. **Phase 0b must ensure TaskScheduler is always registered in the service provider.**

**Impact for Your Phase 0b PR:**
- These tests form the safety net for your code changes
- If constructor signatures change or service resolution breaks, tests catch it immediately
- TaskScheduler must be registered in App.xaml.cs before TopLevelCommandManager is instantiated
- Test files available at: `src/modules/cmdpal/Tests/Microsoft.CmdPal.UI.ViewModels.UnitTests/`

---



## 2026-04-24: Phase 0b DI Migration - IServiceProvider Elimination

**Task:** Eliminate IServiceProvider from ViewModel layer (TopLevelCommandManager, TopLevelViewModel, CommandProviderWrapper)
**Status:** ✅ COMPLETE

**Files Modified:**
- Microsoft.CmdPal.UI.ViewModels\TopLevelCommandManager.cs - Constructor now accepts 8 specific dependencies instead of IServiceProvider
- Microsoft.CmdPal.UI.ViewModels\CommandProviderWrapper.cs - All 5 methods (LoadTopLevelCommands, InitializeCommands, PinCommand, UnpinCommand, PinDockBand, UnpinDockBand) refactored to accept specific services
- Microsoft.CmdPal.UI.ViewModels\TopLevelViewModel.cs - Constructor now accepts HotkeyManager, AliasManager, ISettingsService directly; IContextMenuFactory made required (non-nullable)
- Microsoft.CmdPal.UI\App.xaml.cs - TopLevelCommandManager registration uses factory lambda to resolve all dependencies
- Tests updated to use direct dependency injection instead of IServiceProvider

**Patterns Discovered:**
1. **Moq Expression Trees**: Cannot use methods with optional parameters in Setup() - must specify all parameters explicitly
2. **StyleCop Tuple Naming**: Tuple element names must use PascalCase
3. **Null Safety in Tests**: HotkeyManager and AliasManager have no parameterless constructors; pass null in tests where they won't be called
4. **Mock Factory Setup**: IContextMenuFactory.UnsafeBuildAndInitMoreCommands must return empty list, not null
5. **Null-Conditional Operators**: Added ?. operators to HotkeyManager and AliasManager usage in TopLevelViewModel for null safety

**Test Results:** All 76 tests passing (Flint's 25 baseline tests + 51 existing tests)

**Gotchas:**
- Extension?.PackageFamilyName was using null-forgiving operator (!) for built-in providers - fixed with ?? operator
- CommandItemViewModel needs IContextMenuFactory mock to return empty list to avoid NullReferenceException
