# Decision: Phase 0a Baseline Test Coverage Complete

**By:** Flint (Tester)  
**Date:** 2026-04-22  
**Status:** ✅ COMPLETE  

## What
Wrote 25 baseline unit tests across 3 test classes capturing the current behavior of `TopLevelCommandManager`, `HotkeyManager`, and `DefaultContextMenuFactory` **before** any DI migration code changes.

## Test Breakdown
- **TopLevelCommandManagerTests** (12 tests): Constructor injection of IServiceProvider + ICommandProviderCache, TaskScheduler resolution, initial state assertions, Dispose, LoadBuiltinsAsync with 0/1/2 ICommandProvider registrations
- **HotkeyManagerTests** (7 tests): Constructor injection, UpdateHotkey add/remove/replace/conflict scenarios via captured transform lambdas
- **DefaultContextMenuFactoryTests** (6 tests): Singleton pattern, IContextMenuFactory implementation, null/empty item handling, no-op method

## Key Finding
The `!` null-forgiving operator on `GetService<TaskScheduler>()!` is compile-time only — the constructor does NOT throw at runtime if TaskScheduler is unregistered. This means Phase 0b must ensure TaskScheduler is always registered.

## Why
These tests form the safety net for Phase 0b. If constructor signatures change during DI migration and something breaks, these tests will catch it immediately.

## Files
- `src/modules/cmdpal/Tests/Microsoft.CmdPal.UI.ViewModels.UnitTests/TopLevelCommandManagerTests.cs`
- `src/modules/cmdpal/Tests/Microsoft.CmdPal.UI.ViewModels.UnitTests/HotkeyManagerTests.cs`
- `src/modules/cmdpal/Tests/Microsoft.CmdPal.UI.ViewModels.UnitTests/DefaultContextMenuFactoryTests.cs`
