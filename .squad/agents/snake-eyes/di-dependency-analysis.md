# DI Dependency Analysis for Command Palette

**Date:** 2025-01-28
**Analyst:** Snake Eyes
**Purpose:** Map ViewModel & Service constructor dependencies to replace `App.Current.Services` with proper constructor injection

---

## Executive Summary

### Key Findings

1. **Service Locator Usage:** Concentrated in 2 locations:
   - `TopLevelCommandManager` uses `IServiceProvider` to resolve `TaskScheduler` and enumerate `ICommandProvider` instances
   - `TopLevelViewModel.Hotkey` setter uses `IServiceProvider.GetService<HotkeyManager>()`
   
2. **No Circular Dependencies Detected** at the ViewModel level between the critical classes
   
3. **Critical Dependency Chains:**
   - `TopLevelCommandManager` → `IServiceProvider` → needs all `ICommandProvider` instances
   - `SettingsViewModel` → `TopLevelCommandManager` + `ISettingsService` + `IThemeService`
   - `DockViewModel` → `TopLevelCommandManager` + `ISettingsService`
   - `ShellViewModel` → `IRootPageService` + `IPageViewModelFactoryService`

4. **Factory Pattern:** `IPageViewModelFactoryService` (impl: `CommandPalettePageViewModelFactory`) creates ViewModels dynamically - this is already a factory and won't need changes

5. **Singleton Pattern:** `DefaultContextMenuFactory.Instance` - static singleton used throughout

---

## 1. ViewModel Constructor Dependencies

### Critical ViewModels

#### ShellViewModel
**Constructor Parameters:**
- `TaskScheduler scheduler`
- `IRootPageService rootPageService`
- `IPageViewModelFactoryService pageViewModelFactory`
- `IAppHostService appHostService`

**Service Locator Calls:** None inside class
**Manual Instantiations:**
- `new NullPageViewModel(_scheduler, appHostService.GetDefaultHost())`
- `new LoadingPageViewModel(null, _scheduler, appHostService.GetDefaultHost())`

**Notes:** Root coordinator for page navigation and loading

---

#### TopLevelCommandManager
**Constructor Parameters:**
- `IServiceProvider serviceProvider` ⚠️
- `ICommandProviderCache commandProviderCache`

**Service Locator Calls:**
```csharp
// In constructor:
_taskScheduler = _serviceProvider.GetService<TaskScheduler>()!;

// In LoadBuiltinsAsync():
var builtInCommands = _serviceProvider.GetServices<ICommandProvider>();
```

**Notes:** 
- Uses `IServiceProvider` to enumerate all registered `ICommandProvider` instances
- This is a legitimate use case for service locator when loading plugins/providers
- **Recommendation:** Keep `IServiceProvider` but document why it's needed (plugin enumeration)

---

#### TopLevelViewModel
**Constructor Parameters:**
- `ISettingsService settingsService`
- `ProviderSettings providerSettings`
- `IServiceProvider serviceProvider` ⚠️
- `CommandItemViewModel commandItemViewModel`
- `IContextMenuFactory contextMenuFactory`

**Service Locator Calls:**
```csharp
// In Hotkey setter:
_serviceProvider.GetService<HotkeyManager>()!.UpdateHotkey(Id, value);
```

**Notes:**
- Uses `IServiceProvider` to get `HotkeyManager` in property setter
- **Recommendation:** Inject `HotkeyManager` in constructor instead

---

#### DockViewModel
**Constructor Parameters:**
- `TopLevelCommandManager tlcManager`
- `IContextMenuFactory contextMenuFactory`
- `TaskScheduler scheduler`
- `ISettingsService settingsService`

**Service Locator Calls:** None
**Manual Instantiations:** None (uses DI cleanly)

---

#### SettingsViewModel
**Constructor Parameters:**
- `TopLevelCommandManager topLevelCommandManager`
- `TaskScheduler scheduler`
- `IThemeService themeService`
- `ISettingsService settingsService`

**Service Locator Calls:** None
**Manual Instantiations:**
- `new AppearanceSettingsViewModel(themeService, settingsService)`
- `new DockAppearanceSettingsViewModel(themeService, settingsService)`
- `new ProviderSettingsViewModel(...)` (in loop)

**Notes:** Creates child ViewModels manually - these could be injected as factories if needed

---

#### MainWindowViewModel
**Constructor Parameters:**
- `IThemeService themeService`

**Service Locator Calls:** None
**Notes:** Simplest ViewModel - only needs theme service

---

#### ListViewModel
**Constructor Parameters:**
- `IListPage model`
- `TaskScheduler scheduler`
- `AppExtensionHost host`
- `ICommandProviderContext providerContext`
- `IContextMenuFactory contextMenuFactory`

**Service Locator Calls:** None
**Manual Instantiations:**
- `new CommandItemViewModel(...)` (creates ViewModels for list items)

---

#### PageViewModel (Base Class)
**Constructor Parameters:**
- `IPage? model`
- `TaskScheduler scheduler`
- `AppExtensionHost extensionHost`
- `ICommandProviderContext providerContext`

**Service Locator Calls:** None
**Base for:** `ListViewModel`, `ContentPageViewModel`, `LoadingPageViewModel`, `NullPageViewModel`

---

#### CommandItemViewModel
**Constructor Parameters:**
- `ExtensionObject<ICommandItem> item`
- `WeakReference<IPageContext> errorContext`
- `IContextMenuFactory? contextMenuFactory`

**Service Locator Calls:** None
**Notes:** Created by `ListViewModel` and other consumers

---

### Supporting ViewModels

#### HotkeyManager
**Constructor Parameters:**
- `TopLevelCommandManager tlcManager`
- `ISettingsService settingsService`

**Service Locator Calls:** None
**Notes:** Currently resolved via service locator in `TopLevelViewModel` - should be injected

---

#### AppearanceSettingsViewModel
**Constructor Parameters:**
- `IThemeService themeService`
- `ISettingsService settingsService`

**Service Locator Calls:** None
**Manual Instantiations:** Creates `UISettings` (WinRT API)

---

#### DockAppearanceSettingsViewModel
**Constructor Parameters:**
- `IThemeService themeService`
- `ISettingsService settingsService`

**Service Locator Calls:** None
**Manual Instantiations:** Creates `UISettings` (WinRT API)

---

#### RecentCommandsManager
**Constructor Parameters:** None (parameterless)
**Service Locator Calls:** None
**Notes:** Pure data class (record type)

---

## 2. Service Layer Dependencies

### Service Interfaces & Implementations

| Interface | Implementation | Project | Constructor Parameters |
|-----------|----------------|---------|------------------------|
| `IAppStateService` | `AppStateService` | Microsoft.CmdPal.UI.ViewModels | `IPersistenceService`, `IApplicationInfoService` |
| `ISettingsService` | `SettingsService` | Microsoft.CmdPal.UI.ViewModels | `IPersistenceService`, `IApplicationInfoService` |
| `IPersistenceService` | `PersistenceService` | Microsoft.CmdPal.UI.ViewModels | None (parameterless) |
| `IThemeService` | *(not in this directory)* | *(likely in UI project)* | *(needs investigation)* |
| `ICommandProviderCache` | `DefaultCommandProviderCache` | Microsoft.CmdPal.UI.ViewModels | None (parameterless) |
| `IExtensionTemplateService` | `ExtensionTemplateService` | Microsoft.CmdPal.UI.ViewModels | None (parameterless constructor, or `string templateZipPath` for testing) |
| `IApplicationInfoService` | `ApplicationInfoService` | Microsoft.CmdPal.Common | None (or optional `Func<string>? getLogDirectory`) |
| `IExtensionService` | *(not found in Services/)* | *(likely in Common or UI project)* | *(needs investigation)* |
| `IContextMenuFactory` | `DefaultContextMenuFactory` | Microsoft.CmdPal.UI.ViewModels | None (private constructor, uses static `Instance`) |
| `IRootPageService` | *(not found)* | *(likely in UI project)* | *(needs investigation)* |
| `IPageViewModelFactoryService` | `CommandPalettePageViewModelFactory` | Microsoft.CmdPal.UI.ViewModels | `TaskScheduler`, `IContextMenuFactory` |

### Service Dependencies

```
AppStateService
├── IPersistenceService (PersistenceService)
└── IApplicationInfoService (ApplicationInfoService)

SettingsService
├── IPersistenceService (PersistenceService)
└── IApplicationInfoService (ApplicationInfoService)

PersistenceService
└── (no dependencies)

ApplicationInfoService
└── (no dependencies)

DefaultCommandProviderCache
└── (no dependencies)

ExtensionTemplateService
└── (no dependencies)
```

**No circular dependencies detected in service layer.**

---

## 3. Cross-Project Dependencies

### Microsoft.CmdPal.UI.ViewModels.csproj
**References:**
- `../extensionsdk/Microsoft.CommandPalette.Extensions.Toolkit/`
- `../ext/Microsoft.CmdPal.Ext.Apps/`
- `../../../common/ManagedCommon/`

**Packages:**
- CommunityToolkit.Mvvm
- AdaptiveCards (ObjectModel & Rendering)
- Microsoft.Windows.CsWin32
- WyHash

---

### Microsoft.CmdPal.Common.csproj
**References:**
- `../extensionsdk/Microsoft.CommandPalette.Extensions.Toolkit/`
- `../../../common/ManagedCommon/`

**Packages:**
- Microsoft.Extensions.Hosting
- Microsoft.Windows.CsWin32
- Microsoft.WindowsAppSDK
- Microsoft.Web.WebView2

---

### Microsoft.CmdPal.UI.csproj
*(Not fully analyzed - view was truncated)*
**Known to reference:**
- Microsoft.CmdPal.UI.ViewModels
- Microsoft.CmdPal.Common

---

## 4. Circular Dependency Analysis

### High-Risk Classes Analyzed

✅ **ShellViewModel** ← no circular refs
- Depends on: `IRootPageService`, `IPageViewModelFactoryService`, `IAppHostService`, `TaskScheduler`
- Used by: UI layer (not by services)

✅ **TopLevelCommandManager** ← no circular refs
- Depends on: `IServiceProvider`, `ICommandProviderCache`
- Used by: `SettingsViewModel`, `DockViewModel`, `HotkeyManager`, `TopLevelViewModel`
- **Note:** Uses `IServiceProvider` to enumerate `ICommandProvider` instances (legitimate plugin enumeration)

✅ **DockViewModel** ← no circular refs
- Depends on: `TopLevelCommandManager`, `IContextMenuFactory`, `TaskScheduler`, `ISettingsService`
- Used by: UI layer

✅ **MainWindowViewModel** ← no circular refs
- Depends on: `IThemeService`
- Used by: UI layer

✅ **TopLevelViewModel** ← potential optimization
- Depends on: `ISettingsService`, `IServiceProvider`, `CommandItemViewModel`, `IContextMenuFactory`
- Uses `IServiceProvider` to get `HotkeyManager` in property setter
- **Risk Level:** Low - but could be improved by injecting `HotkeyManager`

### Dependency Graph (Simplified)

```
┌─────────────────────┐
│   ShellViewModel    │
└──────┬──────────────┘
       │
       ├──► IRootPageService
       ├──► IPageViewModelFactoryService ──► CommandPalettePageViewModelFactory
       │                                      ├──► TaskScheduler
       │                                      └──► IContextMenuFactory
       ├──► IAppHostService
       └──► TaskScheduler

┌──────────────────────────┐
│ TopLevelCommandManager   │
└──────┬───────────────────┘
       │
       ├──► IServiceProvider (plugin enumeration)
       │     └──► ICommandProvider (multiple instances)
       └──► ICommandProviderCache

┌─────────────────────┐
│  SettingsViewModel  │
└──────┬──────────────┘
       │
       ├──► TopLevelCommandManager
       ├──► TaskScheduler
       ├──► IThemeService
       └──► ISettingsService

┌─────────────────────┐
│    DockViewModel    │
└──────┬──────────────┘
       │
       ├──► TopLevelCommandManager
       ├──► IContextMenuFactory
       ├──► TaskScheduler
       └──► ISettingsService

┌─────────────────────┐
│ TopLevelViewModel   │
└──────┬──────────────┘
       │
       ├──► ISettingsService
       ├──► IServiceProvider (only for HotkeyManager lookup)
       ├──► CommandItemViewModel
       └──► IContextMenuFactory
```

**No circular dependencies detected.** All dependencies flow in one direction.

---

## 5. Factory and Provider Patterns

### IPageViewModelFactoryService
**Implementation:** `CommandPalettePageViewModelFactory`
**Purpose:** Creates appropriate ViewModel for a given `IPage` instance
**Constructor:**
```csharp
CommandPalettePageViewModelFactory(TaskScheduler scheduler, IContextMenuFactory contextMenuFactory)
```
**Factory Method:**
```csharp
PageViewModel? TryCreatePageViewModel(IPage page, bool nested, AppExtensionHost host, ICommandProviderContext providerContext)
```
**Returns:**
- `ListViewModel` for `IListPage` / `MainListPage`
- `CommandPaletteContentPageViewModel` for `IContentPage`
- `null` for unknown page types

**DI Impact:** Already a factory - no changes needed

---

### IContextMenuFactory
**Implementation:** `DefaultContextMenuFactory`
**Pattern:** Static singleton (`DefaultContextMenuFactory.Instance`)
**Constructor:** Private (singleton)
**Methods:**
- `List<IContextItemViewModel> UnsafeBuildAndInitMoreCommands(...)`
- `void AddMoreCommandsToTopLevel(...)`

**DI Impact:** 
- Currently static singleton
- **Recommendation:** Register as singleton in DI container instead of static field
- Update all `DefaultContextMenuFactory.Instance` references to use injected instance

---

### ICommandProviderCache
**Implementation:** `DefaultCommandProviderCache`
**Constructor:** Parameterless
**Purpose:** Caches command provider metadata to disk
**Methods:**
- `void Memorize(string providerId, CommandProviderCacheItem item)`
- `CommandProviderCacheItem? Recall(string providerId)`

**DI Impact:** Can be registered as singleton

---

### IExtensionService
**Location:** `Microsoft.CmdPal.Common.Services`
**Implementation:** Not found in Services/ directory (likely in another file)
**Purpose:** Manages extension lifecycle (start, stop, enumerate)
**Methods:**
- `Task<IEnumerable<IExtensionWrapper>> GetInstalledExtensionsAsync(...)`
- `void EnableExtension(string extensionUniqueId)`
- `void DisableExtension(string extensionUniqueId)`

**DI Impact:** Needs investigation to find implementation

---

## 6. Recommendations for DI Migration

### Phase 1: Quick Wins (No Breaking Changes)

1. **Inject `HotkeyManager` into `TopLevelViewModel`**
   ```csharp
   // Current:
   public TopLevelViewModel(ISettingsService, ProviderSettings, IServiceProvider, CommandItemViewModel, IContextMenuFactory)
   
   // Recommended:
   public TopLevelViewModel(ISettingsService, ProviderSettings, CommandItemViewModel, IContextMenuFactory, HotkeyManager)
   ```
   - Remove `IServiceProvider` parameter
   - Remove `_serviceProvider.GetService<HotkeyManager>()` call in `Hotkey` setter
   - Store `HotkeyManager` as a field

2. **Replace `DefaultContextMenuFactory.Instance` with DI**
   - Register `DefaultContextMenuFactory` as singleton in DI container
   - Update all `DefaultContextMenuFactory.Instance` references to use constructor injection
   - Remove static `Instance` field

3. **Document `IServiceProvider` usage in `TopLevelCommandManager`**
   - Add XML doc comment explaining plugin enumeration use case
   - This is a legitimate use of service locator pattern for plugin systems

### Phase 2: Service Layer Cleanup

4. **Register all services as singletons:**
   ```csharp
   services.AddSingleton<IPersistenceService, PersistenceService>();
   services.AddSingleton<IApplicationInfoService, ApplicationInfoService>();
   services.AddSingleton<IAppStateService, AppStateService>();
   services.AddSingleton<ISettingsService, SettingsService>();
   services.AddSingleton<ICommandProviderCache, DefaultCommandProviderCache>();
   services.AddSingleton<IExtensionTemplateService, ExtensionTemplateService>();
   services.AddSingleton<IContextMenuFactory, DefaultContextMenuFactory>();
   services.AddSingleton<IPageViewModelFactoryService, CommandPalettePageViewModelFactory>();
   services.AddSingleton<HotkeyManager>();
   services.AddSingleton<TopLevelCommandManager>();
   ```

### Phase 3: ViewModel Registration

5. **Register ViewModels with proper scopes:**
   ```csharp
   // Singletons (live for app lifetime)
   services.AddSingleton<ShellViewModel>();
   services.AddSingleton<MainWindowViewModel>();
   services.AddSingleton<SettingsViewModel>();
   services.AddSingleton<DockViewModel>();
   
   // Transients (created per-use)
   services.AddTransient<ListViewModel>();
   services.AddTransient<PageViewModel>();
   services.AddTransient<CommandItemViewModel>();
   services.AddTransient<TopLevelViewModel>();
   ```

### Phase 4: UI Layer Integration

6. **Replace `App.Current.Services` calls in UI layer**
   - Audit all `.xaml.cs` files (12 found with `App.Current.Services`)
   - Pass services through constructor or use DI-aware code-behind patterns
   - Consider using XAML markup extensions for ViewModel binding

---

## 7. Risk Assessment

### Low Risk
✅ Service layer (PersistenceService, ApplicationInfoService, etc.) - no circular deps
✅ MainWindowViewModel - simple single dependency
✅ Factory patterns already in place

### Medium Risk
⚠️ TopLevelCommandManager - uses `IServiceProvider` for plugin enumeration (keep as-is)
⚠️ SettingsViewModel - manually creates child ViewModels (could inject factories)

### High Risk
🔴 UI layer (.xaml.cs files) - needs careful audit of `App.Current.Services` usage
🔴 IThemeService implementation - not found in Services/ (needs investigation)
🔴 IRootPageService implementation - not found (needs investigation)

---

## 8. Missing Pieces (Needs Investigation)

1. **IThemeService implementation** - interface exists, but implementation not found in Services/
2. **IRootPageService implementation** - interface exists, no implementation found
3. **IExtensionService implementation** - interface exists in Common.Services, implementation not found
4. **IAppHostService** - referenced in ShellViewModel, not found in Services/
5. **Complete UI layer analysis** - MainWindow.xaml.cs and other .xaml.cs files truncated

---

## 9. Files Audited

### ViewModels (Microsoft.CmdPal.UI.ViewModels)
- ✅ ShellViewModel.cs
- ✅ MainWindowViewModel.cs
- ✅ TopLevelCommandManager.cs
- ✅ TopLevelViewModel.cs
- ✅ SettingsViewModel.cs
- ✅ DockViewModel.cs
- ✅ ListViewModel.cs
- ✅ PageViewModel.cs
- ✅ CommandItemViewModel.cs
- ✅ AppearanceSettingsViewModel.cs
- ✅ DockAppearanceSettingsViewModel.cs
- ✅ HotkeyManager.cs
- ✅ RecentCommandsManager.cs
- ✅ CommandPalettePageViewModelFactory.cs
- ✅ DefaultContextMenuFactory.cs

### Services (Microsoft.CmdPal.UI.ViewModels/Services)
- ✅ AppStateService.cs / IAppStateService.cs
- ✅ SettingsService.cs / ISettingsService.cs
- ✅ PersistenceService.cs / IPersistenceService.cs
- ✅ ExtensionTemplateService.cs / IExtensionTemplateService.cs
- ✅ DefaultCommandProviderCache.cs / ICommandProviderCache.cs
- ✅ IThemeService.cs (interface only)

### Services (Microsoft.CmdPal.Common/Services)
- ✅ ApplicationInfoService.cs / IApplicationInfoService.cs
- ✅ IExtensionService.cs (interface only)
- ✅ IExtensionWrapper.cs (interface only)
- ✅ IRunHistoryService.cs (interface only)

### Other
- ✅ IRootPageService.cs (interface only)
- ✅ IContextMenuFactory.cs

---

## Summary Statistics

| Metric | Count |
|--------|-------|
| ViewModels Analyzed | 15 |
| Services Analyzed | 11 |
| Service Interfaces | 13 |
| Classes with `IServiceProvider` dependency | 2 |
| Classes with `App.Current.Services` calls | 0 (in ViewModels layer) |
| Circular dependencies found | 0 |
| Singleton patterns (static) | 1 (`DefaultContextMenuFactory`) |
| Factory patterns | 2 (`IPageViewModelFactoryService`, `IContextMenuFactory`) |

---

## Appendix: Full Constructor Signature Reference

```csharp
// Core ViewModels
ShellViewModel(TaskScheduler, IRootPageService, IPageViewModelFactoryService, IAppHostService)
MainWindowViewModel(IThemeService)
TopLevelCommandManager(IServiceProvider, ICommandProviderCache)
TopLevelViewModel(ISettingsService, ProviderSettings, IServiceProvider, CommandItemViewModel, IContextMenuFactory)
DockViewModel(TopLevelCommandManager, IContextMenuFactory, TaskScheduler, ISettingsService)
SettingsViewModel(TopLevelCommandManager, TaskScheduler, IThemeService, ISettingsService)
ListViewModel(IListPage, TaskScheduler, AppExtensionHost, ICommandProviderContext, IContextMenuFactory)
PageViewModel(IPage?, TaskScheduler, AppExtensionHost, ICommandProviderContext)
CommandItemViewModel(ExtensionObject<ICommandItem>, WeakReference<IPageContext>, IContextMenuFactory?)

// Supporting ViewModels
HotkeyManager(TopLevelCommandManager, ISettingsService)
AppearanceSettingsViewModel(IThemeService, ISettingsService)
DockAppearanceSettingsViewModel(IThemeService, ISettingsService)
RecentCommandsManager() // parameterless

// Services
AppStateService(IPersistenceService, IApplicationInfoService)
SettingsService(IPersistenceService, IApplicationInfoService)
PersistenceService() // parameterless
ApplicationInfoService() // parameterless or (Func<string>? getLogDirectory)
DefaultCommandProviderCache() // parameterless
ExtensionTemplateService() // parameterless or (string templateZipPath)

// Factories
CommandPalettePageViewModelFactory(TaskScheduler, IContextMenuFactory)
DefaultContextMenuFactory() // private constructor, static Instance
```
