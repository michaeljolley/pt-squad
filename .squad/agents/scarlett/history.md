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

## 2026-04-23: Phase 0a Baseline Tests Complete (Flint)

**Status:** ✅ COMPLETE  
**Update by Scribe:** Baseline safety net established.

Flint completed 25 unit tests for Phase 0 critical classes:
- **TopLevelCommandManagerTests:** 12 tests ✅
- **HotkeyManagerTests:** 7 tests ✅
- **DefaultContextMenuFactoryTests:** 6 tests ✅

All tests passing. Ready for Snake Eyes Phase 0b implementation.

**Key Finding:** `GetService<TaskScheduler>()!` null-forgiving operator is compile-time only. At runtime, constructor may store null if unregistered. Phase 0b must ensure TaskScheduler is always registered.

---

## 2026-04-23: Phase 0b DI Quick Wins Complete (Snake Eyes + Coordinator)

**Status:** ✅ COMPLETE  
**Update by Scribe:** ViewModel layer now 100% free of IServiceProvider.

Snake Eyes successfully eliminated `IServiceProvider` from entire ViewModel layer with support from Coordinator (fix) to resolve circular dependency.

### What Changed
1. **TopLevelViewModel:** HotkeyManager and AliasManager now injected (removed service locator calls from Hotkey property and AliasText setter)
2. **DefaultContextMenuFactory:** Converted from static instance to DI-based singleton, registered as `IContextMenuFactory`
3. **TopLevelCommandManager:** Constructor refactored to accept 8 specific dependencies instead of `IServiceProvider`
4. **CommandProviderWrapper:** Methods refactored to accept specific services

### Critical Fix: Lazy<T> for Circular Dependencies
- **Problem:** TopLevelCommandManager needs HotkeyManager and AliasManager, but both depend on TopLevelCommandManager
- **Solution:** Inject `Lazy<HotkeyManager>` and `Lazy<AliasManager>` to defer resolution until first use
- **Outcome:** Zero runtime circular dependency issues, verified by all tests passing

### Test Results
- ✅ 76 tests passing (25 baseline Phase 0a + 51 updated/existing Phase 0b)
- ✅ ViewModels project builds cleanly
- ✅ Zero circular dependencies confirmed
- ✅ All IServiceProvider references removed from ViewModel layer

### Files Changed
**Production:** TopLevelCommandManager.cs, TopLevelViewModel.cs, CommandProviderWrapper.cs, App.xaml.cs (4 files)
**Tests:** TopLevelCommandManagerTests.cs, HotkeyManagerTests.cs, DefaultContextMenuFactoryTests.cs (3 files)

### Impact
- Service locator calls eliminated: ~10
- Improved testability and DI clarity
- Phase 1 UI Layer Quick Wins ready to begin

**NOTE FOR PHASE 1:** Phase 0b Lazy<T> pattern may be useful reference for UI layer property injection strategy. Coordinator's circular dependency resolution technique is proven.

---

