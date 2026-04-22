# Project Context

- **Owner:** Michael Jolley
- **Project:** Microsoft Command Palette (CmdPal) — a PowerToys utility providing a command launcher with extensible plugin architecture
- **Stack:** C# (WinUI 3 / WinAppSDK), C++/WinRT, XAML, WinRT IDL
- **Scope:** `src/modules/cmdpal/` and `CommandPalette.slnf` only
- **Created:** 2026-04-20

## Learnings

### DI Dependency Analysis (2025-01-28)

**Task:** Analyzed ViewModel and Service layer constructor dependencies to prepare for replacing `App.Current.Services` service locator with proper constructor injection.

**Key Findings:**
1. **Service Locator Usage (Minimal):**
   - `TopLevelCommandManager` uses `IServiceProvider` to enumerate all `ICommandProvider` instances (legitimate plugin enumeration pattern)
   - `TopLevelViewModel.Hotkey` setter uses `IServiceProvider.GetService<HotkeyManager>()` (can be refactored to constructor injection)
   
2. **No Circular Dependencies:** Analyzed 15 ViewModels and 11 services - all dependencies flow in one direction
   
3. **Critical Dependency Chains:**
   - `ShellViewModel` → `IRootPageService`, `IPageViewModelFactoryService`, `IAppHostService`, `TaskScheduler`
   - `TopLevelCommandManager` → `IServiceProvider` (for plugin enumeration), `ICommandProviderCache`
   - `SettingsViewModel` → `TopLevelCommandManager`, `ISettingsService`, `IThemeService`, `TaskScheduler`
   - `DockViewModel` → `TopLevelCommandManager`, `ISettingsService`, `IContextMenuFactory`, `TaskScheduler`
   
4. **Service Layer Clean:** All services have simple dependency chains:
   - `AppStateService` → `IPersistenceService`, `IApplicationInfoService`
   - `SettingsService` → `IPersistenceService`, `IApplicationInfoService`
   - `PersistenceService` → (no dependencies)
   - `ApplicationInfoService` → (no dependencies)
   
5. **Factory Patterns:**
   - `IPageViewModelFactoryService` (impl: `CommandPalettePageViewModelFactory`) - creates ViewModels dynamically, already DI-friendly
   - `IContextMenuFactory` (impl: `DefaultContextMenuFactory`) - uses static singleton pattern (`Instance`) - should migrate to DI singleton
   
6. **Recommendations:**
   - **Quick Win:** Inject `HotkeyManager` into `TopLevelViewModel` constructor, remove `IServiceProvider` usage in property setter
   - **Quick Win:** Replace `DefaultContextMenuFactory.Instance` with DI-registered singleton
   - **Keep:** `TopLevelCommandManager`'s use of `IServiceProvider` for plugin enumeration (legitimate pattern)
   - **Phase 2:** Register all services as singletons in DI container
   - **Phase 3:** Audit UI layer (.xaml.cs files) for `App.Current.Services` usage (12 files found)

**Constructor Patterns Observed:**
- ViewModels typically depend on: `TaskScheduler`, services (ISettingsService, IThemeService), other ViewModels/managers
- Services typically depend on: other services (IPersistenceService, IApplicationInfoService), no ViewModels
- Clean separation between ViewModel layer and Service layer
- No ViewModel-to-Service-to-ViewModel circular dependencies

**Missing Pieces (Need Investigation):**
- `IThemeService` implementation not found in Services/ directory
- `IRootPageService` implementation not found
- `IExtensionService` implementation not found
- `IAppHostService` not found

**Detailed Report:** `C:\sources\PowerToys\.squad\agents\snake-eyes\di-dependency-analysis.md`

---

### Extension System Architecture (2026-04-20 Review)

**Core Extension SDK (WinRT IDL Contracts)**
- **Contract:** `Microsoft.CommandPalette.Extensions.ExtensionsContract (v1)` defined in `src/modules/cmdpal/extensionsdk/Microsoft.CommandPalette.Extensions/Microsoft.CommandPalette.Extensions.idl`
- **Key Interfaces:**
  - `IExtension` (entry point) → `GetProvider(ProviderType)` + `Dispose()`
  - `ICommandProvider` (base) → `TopLevelCommands()`, `FallbackCommands()`, `GetCommand(id)`, `InitializeWithHost()`
  - `ICommandProvider2` → `GetApiExtensionStubs()` (WinRT type cache population)
  - `ICommandProvider3` → `GetDockBands()` (toolbar strips)
  - `ICommandProvider4` → `GetCommandItem(id)` (by-ID lookup)
- **Navigation Model:** `CommandResultKind` enum (Dismiss, GoHome, GoBack, Hide, KeepOpen, GoToPage, ShowToast, Confirm)
- **UI Elements:**
  - `IListPage` (with `IDynamicListPage` for runtime filtering)
  - `IContentPage` (generic content with details pane)
  - `IListItem` (with tags, details, sections, text-to-suggest)
  - Grid layouts: `ISmallGridLayout`, `IMediumGridLayout`, `IGalleryGridLayout`
  - Content types: `IFormContent`, `IMarkdownContent`, `IPlainTextContent`, `IImageContent`, `ITreeContent`
- **Status/Logging:** `IExtensionHost` → `ShowStatus()`, `HideStatus()`, `LogMessage()` with `IStatusMessage` progress tracking
- **Command Binding:** `IInvokableCommand` (executable), `ICommandContextItem` (context menu items), `IFallbackCommandItem`/`IFallbackCommandItem2` (fallback handlers)

**Extension Toolkit (C# Helpers)**
Location: `src/modules/cmdpal/extensionsdk/Microsoft.CommandPalette.Extensions.Toolkit/`
- **Base Classes:**
  - `CommandProvider` (abstract) → implements all 4 ICommandProvider* interfaces
  - `CommandItem` → implements `ICommandItem`, property change tracking via `WeakEventListener`
  - `ListItem` → extends `ICommandItem` with tags, details, sections
  - `InvokableCommand` → implements `IInvokableCommand` for executable commands
  - `ListPage`/`DynamicListPage` → implements `IListPage` with item loading
  - `ContentPage` → implements `IContentPage` with flexible content models
- **Data Models:**
  - `IconInfo` / `IconData` (light/dark theme support)
  - `Tag` (colored badges with optional icons)
  - `Details` / `DetailsElement` / `DetailsTags` / `DetailsLink` / `DetailsCommands` / `DetailsSeparator`
  - `Filter` / `Filters` (for list filtering)
  - `ProgressState` / `StatusMessage` (async operation tracking)
  - `LogMessage` with `MessageState` (Info, Success, Warning, Error)
- **Navigation:** `GoToPageArgs`, `ToastArgs`, `ConfirmationArgs` (all `ICommandResultArgs`)
- **Observable Utilities:**
  - `BaseObservable` (INotifyPropertyChanged implementation)
  - `INotifyPropChanged` / `IPropChangedEventArgs` (extension-facing)
  - `INotifyItemsChanged` / `IItemsChangedEventArgs` (for dynamic collections)
- **Settings:** `ICommandSettings` with `SettingsPage` → `IContentPage`
- **Special:** `WeakEventListener<TSource, TEventArgs>` for memory-safe event binding

**Built-in Extensions (17 in ext/)**
All extend `CommandProvider` from Toolkit. Common patterns:
1. **Apps/Programs:**
   - `Microsoft.CmdPal.Ext.Apps` (AllAppsCommandProvider) → enumerate UWP + Win32 apps, launch with icon
   - `Microsoft.CmdPal.Ext.WinGet` → package manager integration
   
2. **Run/Shell/System:**
   - `Microsoft.CmdPal.Ext.Shell` (ShellCommandsProvider) → fallback executor with history, integrates `IRunHistoryService`
   - `Microsoft.CmdPal.Ext.System` → system info, controls
   - `Microsoft.CmdPal.Ext.Registry` → registry viewer
   - `Microsoft.CmdPal.Ext.WindowsServices` → service manager
   - `Microsoft.CmdPal.Ext.TimeDate` → date/time/calendar
   
3. **Tools/Utilities:**
   - `Microsoft.CmdPal.Ext.Calc` → calculator
   - `Microsoft.CmdPal.Ext.WebSearch` → web search fallback
   - `Microsoft.CmdPal.Ext.ClipboardHistory` → clipboard manager
   - `Microsoft.CmdPal.Ext.Bookmark` → bookmarks
   - `Microsoft.CmdPal.Ext.WindowWalker` → window switcher
   
4. **Terminal/PowerToys:**
   - `Microsoft.CmdPal.Ext.WindowsTerminal` → terminal integration
   - `Microsoft.CmdPal.Ext.RemoteDesktop` → RDP connections
   - `Microsoft.CmdPal.Ext.PowerToys` → launch other PowerToys modules
   - `Microsoft.CmdPal.Ext.Indexer` → Windows Search integration
   - `Microsoft.CmdPal.Ext.PerformanceMonitor` → perf metrics
   
5. **Samples:**
   - `SamplePagesExtension` → demonstrates all SDK features
   - `ProcessMonitorExtension` → real-world example

**Extension Loading (TopLevelCommandManager)**
Location: `src/modules/cmdpal/Microsoft.CmdPal.UI.ViewModels/TopLevelCommandManager.cs`
- Built-ins: `IServiceProvider.GetServices<ICommandProvider>()` → synchronous in-proc, loaded first
- Extensions: Must be discovered separately (assembly scanning at startup or config-driven)
- `CommandProviderWrapper` manages each provider with timeouts:
  - `ExtensionStartTimeout` = 10s
  - `CommandLoadTimeout` = 10s
  - `BackgroundStartTimeout` = 60s (background extensions)
- `LoadTopLevelCommands()` extracts top-level items + fallback items + dock bands
- `ReloadAllCommandsAsyncCore` uses `SupersedingAsyncGate` to coalesce reload requests
- **Thread Safety:** Lock hierarchy: `_commandProvidersLock` > `_dockBandsLock` to prevent deadlocks

**Native C++ Components**

1. **CmdPalModuleInterface** (`src/modules/cmdpal/CmdPalModuleInterface/`)
   - Implements `PowertoyModuleIface` contract (from common interface)
   - `enable()` → Registers + launches CmdPal as MSIX package (Microsoft.CommandPalette or Dev variant)
   - `disable()` → Terminates UI processes
   - `RetryLaunch()` → Exponential backoff (up to 9 retries, 2^9-1 seconds max)
   - `TerminateCmdPal()` → Finds/kills Microsoft.CmdPal.UI.exe processes, waits 1.5s for graceful shutdown
   - Uses `CommonSharedConstants::CMDPAL_EXIT_EVENT` for IPC termination signal

2. **CmdPalKeyboardService** (`src/modules/cmdpal/CmdPalKeyboardService/`)
   - Low-level keyboard hook (WinRT C++/WinRT)
   - `KeyboardListener` class (IDL: `KeyboardListener.idl`)
     - `Start()` / `Stop()` → Install/uninstall global hook via `SetWindowsHookEx(WH_KEYBOARD_LL)`
     - `SetHotkeyAction(win, ctrl, shift, alt, vkey, id)` → Register hotkey
     - `ClearHotkey(id)` / `ClearHotkeys()` → Unregister
     - `SetProcessCommand(callback)` → Set callback (`ProcessCommand` delegate)
   - Thread-safe: `std::multiset<HotkeyDescriptor>` + `std::mutex`
   - Static callback: `LowLevelKeyboardProc` → `DoLowLevelKeyboardProc` (instance method via static instance pointer)
   - Stateful: `vkCodePressed` to track pressed key, `VK_DISABLED` (0x100) to suppress keys

3. **Microsoft.Terminal.UI** (`src/modules/cmdpal/Microsoft.Terminal.UI/`)
   - UI helpers for terminal-like rendering + converters
   - IDL components:
     - `RunHistory.idl` → command history storage
     - `ResourceString.idl` → localized strings
     - `IDirectKeyListener.idl` → keyboard listener interface
     - `FontIconGlyphClassifier.idl` → glyph classification for icons
     - `IconPathConverter.idl` → URI → bitmap conversion
     - `Converters.idl` → WinUI value converters
   - C++ implementation with helpers for rendering pipeline

**API Stability & Versioning**
- **Contract Model:** All IDL uses `[contractversion(1)]` + `[contract(..., 1)]` attribute
- **Versioning Strategy:** `ICommandProvider` → `ICommandProvider2` → `ICommandProvider3` → `ICommandProvider4` (interface inheritance chain for backward compatibility)
- **No Breaking Changes:** New methods added via `ICommandProvider2/3/4` inheritance, not modifications to base
- **Type Cache Workaround:** `GetApiExtensionStubs()` marshals stub objects to populate WinRT type cache (CsWinRT limitation)
- **ABI Stability:** UUIDs assigned to base interfaces to lock ABI: `ICommandResultArgs`, `IFilterItem`, `IContextItem`, `IGridProperties`, `ISmallGridLayout`, `IContent`, `IDetailsSeparator`, `ISeparatorContextItem`

**Strengths**
1. **Clean WinRT abstraction:** Language-agnostic, out-of-proc safe, supports C# + C++/WinRT
2. **Composable UI:** Content pages, grid layouts, markdown, forms all work together
3. **Good defaults:** Toolkit provides sensible base classes (CommandProvider, CommandItem, ListPage)
4. **Async-aware:** Status messages, log messages, form submission all async
5. **Observable pattern:** Property/items change events propagate correctly across ABI boundary
6. **Timeout protection:** Extension load timeouts prevent UI hang
7. **Keyboard service:** Low-level hook in separate DLL, hotkeyable

**Fragile Points / Concerns**
1. **Type Cache Trick:** `GetApiExtensionStubs()` returns stub objects just to populate type cache — fragile, undocumented workaround for CsWinRT limitation. If cache logic changes, extensions break.
2. **Extension Discovery:** No clear documentation on how out-of-proc extensions are discovered. Appears to require external mechanism (config file? assembly scan? plugin folders?).
3. **Contract Versioning:** Relies on interface inheritance chain (v1, v2, v3, v4). If v5 is needed, all providers must update method signatures — no graceful deprecation.
4. **Timeout Handling:** Hard-coded timeouts (10s, 60s). Extensions that legitimately take longer will be cancelled silently.
5. **WeakEventListener:** Custom weak event implementation (`WeakEventListener<TSource, TSender, TArgs>`). Non-standard, maintenance burden.
6. **IExtensionHost Leakage:** `ExtensionHost.Initialize(host)` is static. If multiple extensions call it, last one wins. Race condition possible.
7. **Settings Persistence:** Extensions can provide `ICommandSettings` → `SettingsPage`, but no clear save/load contract. Where are settings stored?
8. **Dock Bands Locking:** Comments warn about deadlock if locks acquired in wrong order. Complex lock hierarchy (CommandProviders > DockBands).
9. **No Deprecation Path:** Once an IDL interface is shipped, removal is a breaking change. V1 contract is immutable forever.
10. **Module Interface Coupling:** CmdPalModuleInterface tightly coupled to MSIX package registration. Hard to test, Windows-specific.

**Key Files by Concern**
- IDL Contracts: `src/modules/cmdpal/extensionsdk/Microsoft.CommandPalette.Extensions/Microsoft.CommandPalette.Extensions.idl`
- Toolkit Base: `src/modules/cmdpal/extensionsdk/Microsoft.CommandPalette.Extensions.Toolkit/CommandProvider.cs`, `CommandItem.cs`, `ListPage.cs`
- Extension Loading: `src/modules/cmdpal/Microsoft.CmdPal.UI.ViewModels/TopLevelCommandManager.cs`
- Module Interface: `src/modules/cmdpal/CmdPalModuleInterface/dllmain.cpp`
- Keyboard Service: `src/modules/cmdpal/CmdPalKeyboardService/KeyboardListener.h`, `KeyboardListener.cpp`
- Sample Extensions: `src/modules/cmdpal/ext/SamplePagesExtension`, `src/modules/cmdpal/ext/ProcessMonitorExtension`

<!-- Append new learnings below. Each entry is something lasting about the project. -->

### ViewModel/Service DI Analysis Summary (2026-04-22)

**Task:** Analyze constructor dependencies across 15 ViewModels and 11 services to identify service locator usage and DI migration opportunities.

**Key Findings:**
- ✅ **Zero circular dependencies** — all dependency chains are acyclic
- ✅ **Minimal service locator usage** — only 2 classes use IServiceProvider:
  1. TopLevelCommandManager — legitimate for plugin enumeration (keep as-is)
  2. TopLevelViewModel — Hotkey property uses GetService<HotkeyManager>() (refactorable)

**Dependency Patterns:**
- **ViewModels:** Depend on TaskScheduler, services (ISettingsService, IThemeService), other ViewModels/managers
- **Services:** Depend on other services (IPersistenceService, IApplicationInfoService), never on ViewModels
- **Clean separation:** No ViewModel-to-Service-to-ViewModel circular dependencies
- **Factory patterns:** IPageViewModelFactoryService (DI-friendly), DefaultContextMenuFactory (static singleton, needs DI migration)

**Quick Wins Identified:**

1. **Inject HotkeyManager into TopLevelViewModel**
   - Eliminates: _serviceProvider.GetService<HotkeyManager>() in Hotkey property setter
   - Change: Add HotkeyManager hotkeyManager constructor parameter
   - Risk: Low, straightforward refactoring
   - Impact: Improves testability, removes one service locator call

2. **Register DefaultContextMenuFactory as DI Singleton**
   - Current: Uses static Instance property accessed throughout codebase
   - Proposed: Register as IContextMenuFactory in DI container, inject via constructor
   - Risk: Low, affects multiple ViewModels but dependencies are clean
   - Impact: Enables factory testing, removes static singleton pattern

**Service Layer Assessment:**
- AppStateService, SettingsService, PersistenceService, ApplicationInfoService have simple, clean dependency chains
- All services are ready for DI container registration
- No migration blockers

**Status:** Service and ViewModel layers are clean and ready for full DI migration. Quick wins can be implemented immediately. No architectural blockers identified.
