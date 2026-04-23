# Coordinator (fix) — Circular Dependency Resolution

**Timestamp:** 2026-04-23T04:40:00Z  
**Agent:** Coordinator (fix)  
**Task:** Fix circular dependency bug  
**Status:** ✅ COMPLETE  

## Mandate

Resolve critical circular dependency bug in Snake Eyes' Phase 0b implementation. Snake Eyes' initial approach had TopLevelCommandManager taking HotkeyManager and AliasManager directly, which would crash at runtime because both depend on TopLevelCommandManager.

## Issue Identified

Snake Eyes' DI registration:
```csharp
// BROKEN: Circular dependency
services.AddSingleton<TopLevelCommandManager>(sp =>
    new TopLevelCommandManager(
        ...,
        sp.GetRequiredService<HotkeyManager>(),
        sp.GetRequiredService<AliasManager>(),
        ...
    )
);

// HotkeyManager and AliasManager both depend on TopLevelCommandManager
services.AddSingleton<HotkeyManager>();  // Tries to resolve TopLevelCommandManager in ctor
services.AddSingleton<AliasManager>();   // Tries to resolve TopLevelCommandManager in ctor
```

This would fail at application startup with a circular dependency exception.

## Solution Implemented

Replace direct dependency with `Lazy<T>` to defer resolution:

1. **TopLevelCommandManager.cs**
   - Changed constructor parameters from `HotkeyManager` and `AliasManager` to `Lazy<HotkeyManager>` and `Lazy<AliasManager>`
   - Updated all usages to access `.Value` only when needed (lazy evaluation breaks the cycle)

2. **App.xaml.cs**
   - Updated DI registration to inject `Lazy<HotkeyManager>` and `Lazy<AliasManager>` instead of direct types
   - All downstream registrations (HotkeyManager, AliasManager) now register cleanly without circular dependency

3. **Test Files**
   - TopLevelCommandManagerTests.cs: Updated mock setup to wrap managers in `new Lazy<T>(mock)`
   - HotkeyManagerTests.cs: Verified tests pass with new lazy injection
   - DefaultContextMenuFactoryTests.cs: Verified tests pass

## How Lazy<T> Breaks the Cycle

- **Before:** TopLevelCommandManager requests HotkeyManager during TopLevelCommandManager construction
- **After:** TopLevelCommandManager requests `Lazy<HotkeyManager>` (a wrapper), not the manager itself
  - The wrapper is constructed instantly (no circular dependency)
  - The actual HotkeyManager is only constructed when `.Value` is first accessed
  - By then, TopLevelCommandManager construction is complete, so the dependency can resolve cleanly

## Test Results

- ✅ All 76 tests passing
- ✅ No circular dependency exceptions at application startup
- ✅ Lazy<T> values evaluated correctly on first use
- ✅ ViewModels project builds cleanly

## Files Modified

- TopLevelCommandManager.cs (Lazy<T> fields)
- App.xaml.cs (Lazy<T> registration)
- TopLevelCommandManagerTests.cs (mock wrapper)
- HotkeyManagerTests.cs (verified)

## Sign-off

Critical bug resolved. Lazy<T> pattern is a proven solution for circular dependency resolution in .NET DI containers. Phase 0b now ready for merge.
