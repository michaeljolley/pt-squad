# Project Context

- **Owner:** Michael Jolley
- **Project:** Microsoft Command Palette (CmdPal) — a PowerToys utility providing a command launcher with extensible plugin architecture
- **Stack:** C# (WinUI 3 / WinAppSDK), C++/WinRT, XAML, WinRT IDL
- **Scope:** `src/modules/cmdpal/` and `CommandPalette.slnf` only
- **Created:** 2026-04-20

## Learnings

### CmdPal Architecture Overview (Architectural Review)

**Overall Structure:**
CmdPal is a layered architecture with clear separation between native/managed boundaries, UI, business logic, and extension SDK:
- **UI Layer** (`Microsoft.CmdPal.UI`): WinUI 3 app, XAML controls, window management
- **ViewModel Layer** (`Microsoft.CmdPal.UI.ViewModels`): MVVM Toolkit-based state management, command providers
- **Common/Shared** (`Microsoft.CmdPal.Common`): Services, logging, IPC bridges, extension loading
- **Extension SDK** (`extensionsdk/Microsoft.CommandPalette.Extensions`): WinRT IDL interfaces + C# toolkit
- **Built-in Extensions** (`ext/`): 17 extensions (Apps, Calc, Clipboard, Shell, System, etc.)
- **Native Components** (`CmdPalKeyboardService`, `CmdPalModuleInterface`, `Microsoft.Terminal.UI`): C++/WinRT

**Dependency Graph (Clean Layering):**
```
UI (Microsoft.CmdPal.UI)
  ↓
ViewModels (Microsoft.CmdPal.UI.ViewModels)
  ↓
Common (Microsoft.CmdPal.Common) + Extension SDK Toolkit
  ↓
Extension SDK IDL (Microsoft.CommandPalette.Extensions.vcxproj)
  ↓
Extensions (ext/) + Native Services (keyboard, module interface)
  ↓
Shared Libraries (ManagedCommon, logger, SettingsAPI, telemetry)
```

**Extension Model & Plug-in Architecture:**
- **IDL Contract** (`extensionsdk/Microsoft.CommandPalette.Extensions/Microsoft.CommandPalette.Extensions.idl`):
  - Core interfaces: `IExtension`, `ICommandProvider`, `ICommand`, `ICommandResult`
  - Navigation enums: `CommandResultKind`, `NavigationMode` (Dismiss, GoHome, GoBack, Hide, KeepOpen, GoToPage, ShowToast, Confirm)
  - Event contracts: `INotifyPropChanged`, `INotifyItemsChanged` for reactive updates
  - Icon & metadata: `IIconInfo`, `IIconData` for themed icons (light/dark)
  
- **C# Toolkit** (`extensionsdk/Microsoft.CommandPalette.Extensions.Toolkit`):
  - Wraps WinRT projections (via CsWinRT)
  - Helper base classes for `ICommandProvider`, `ICommand` implementations
  - Utility types for ToolGood.Words.Pinyin (for search ranking)
  
- **Extension Loading** (via `CommandProviderWrapper` & `AppExtensionHost`):
  - Extensions loaded dynamically via `IExtensionWrapper` (in-proc COM objects)
  - `CommandProviderWrapper` acts as adapter between extension and palette
  - Each extension gets its own `CommandPaletteHost` instance (implements `IExtensionHost`)
  - Extensions communicate back via host methods (logging, status messages, navigation)

**Native/Managed Boundary:**
- **CmdPalKeyboardService** (C++/WinRT): Keyboard hook service (`KeyboardListener.idl`), accessed via CsWinRT projections in UI
- **Microsoft.Terminal.UI** (C++/WinRT): Terminal rendering, converters (icon path, run history), font glyph classification
  - Projected as `.winmd` metadata consumed by C# via `CsWinRTInputs` in `.csproj`
  - Output DLL copied to UI bin folder (`PreserveNewest`)
- **CmdPalModuleInterface** (C++): Implements `PowertoyModuleIface` from `src/common/interface/powertoy_module_interface.h`
  - Entry point for PowerToys Runner, handles module lifecycle (enable/disable)
  - Launches the C# UI (`Microsoft.CmdPal.UI`) as subprocess
  - Links against `logger.vcxproj` and `SettingsAPI.vcxproj` from common

**IPC & Runner Communication:**
- **Module Interface**: C++ module implements PowerToys standard interface (`powertoy_module_interface.h`)
- **Process Model**: CmdPal runs as separate WinUI 3 packaged app (Desktop Bridge)
- **Settings Bridge**: Via common SettingsAPI (for reading GPO, user prefs)
- **Logging**: `ManagedCommon.Logger` + spdlog for C++ components

**MVVM & Dependency Injection:**
- **MVVM Toolkit**: Used for `ObservableObject`, reactive properties, commands
- **Dependency Injection** (Microsoft.Extensions.DependencyInjection):
  - `Program.cs` sets up DI container (Services.AddScoped/Singleton)
  - TaskScheduler passed to ViewModels for thread marshalling
  - Services include: ExtensionService, RunHistoryService, SettingsService
- **Messaging** (MVVM Community Toolkit Messenger):
  - `MainWindow` implements `IRecipient<DismissMessage>`, etc.
  - Types: `DismissMessage`, `ShowWindowMessage`, `ShowPaletteAtMessage` in `Messages/` folder

**Project File Organization:**
- **CmdPal.slnf** references all 60+ projects (shared + cmdpal-specific)
- **CmdPal.pre.props**, **CmdPal.Branding.props**: Shared configuration across UI & ViewModels
- **Common.ExtDependencies.props**: Shared extension deps (Toolkit, Calc engine WinMD, etc.)
- **AoT Support**: LangVersion=preview, PublishAot=true configured in UI project for trimming
- **CsWinRT Interop**: Each C#/C++ boundary uses explicit `CsWinRTInputs` + `.winmd` copying

**Key Design Patterns:**
1. **Host Adapter** (`CommandPaletteHost` + `AppExtensionHost`): Extensions see palette as `IExtensionHost` abstraction
2. **Provider Pattern** (`ICommandProvider` implementations): Each ext returns commands via provider query
3. **Result Pattern** (`ICommandResult` + result args): Decoupled action outcomes from execution
4. **Icon Virtualization** (`IIconInfo`): Lazy-load icons with light/dark variants
5. **Status/Toast Pattern**: Async callbacks for non-blocking user feedback
6. **Logging Observatory**: `GlobalLogPageContext` captures extension logs for debug page

**Strengths:**
- Clean IDL boundary makes extensions language-agnostic (any WinRT-compatible language works)
- MVVM + DI enables testability (16 test projects, unit test coverage on extensions & ViewModels)
- Modular extension loading with in-proc COM hosting (fast, isolated failure domains)
- AoT-ready architecture (PublishAot enabled) for future deployment as single executable

### Dependency Injection Architecture Analysis (2025-01-31)

**Task:** Map full dependency graph and identify circular reference risks for migrating from `App.Current.Services` service locator to constructor injection.

**Findings:**
- ✅ **No circular dependencies** in the service registration graph across all 30+ registered services
- ✅ **Clean layer separation** - ViewModels layer does NOT use `App.Current.Services` anywhere
- ⚠️ **13 XAML pages + 4 controls** use service locator in constructors (WinUI limitation)
- ✅ **All services have interface abstractions** - ready for DI migration

**Service Registration Structure:**
```
App.ConfigureServices:
  - Root: TaskScheduler (from SynchronizationContext), Logging (extension method)
  - AddBuiltInCommands: 17 ICommandProvider singletons (AllApps, Shell, Calculator, etc.)
  - AddCoreServices: ExtensionService, RunHistoryService, RootPageService, ShellViewModel, DockViewModel, factories
  - AddUIServices: PersistenceService, SettingsService, AppStateService, ThemeService, TopLevelCommandManager, Icon services
```

**Dependency Hierarchy (4 levels, bottom-up):**
- **Level 0** (leaf): PersistenceService, ApplicationInfoService, TaskScheduler, all command providers
- **Level 1**: SettingsService, AppStateService, ThemeService (depend on Level 0)
- **Level 2**: RunHistoryService, TopLevelCommandManager, TrayIconService (depend on 0-1)
- **Level 3**: AliasManager, HotkeyManager, DockViewModel (depend on 0-2)
- **Level 4** (top): PowerToysRootPageService, ShellViewModel (depend on 0-3)

**Key Service Dependencies:**
- `TopLevelCommandManager` takes `IServiceProvider` to lazy-resolve `IEnumerable<ICommandProvider>` (not a cycle)
- `PowerToysRootPageService` has 5 constructor params: TLC, AliasManager, FuzzyMatcher, Settings, AppState
- `ShellViewModel` has 4 constructor params: Scheduler, RootPageService, PageFactory, AppHost
- `ThemeService` depends on `ResourceSwapper` + `ISettingsService`

**XAML Instantiation Challenge:**
- 13 pages: MainWindow, GeneralPage, ExtensionsPage, AppearancePage, DockSettingsPage, InternalPage, ShellPage, ListPage, DockWindow, and others
- 4 controls: SearchBar, ContextMenu, FallbackRanker
- All instantiated by WinUI XAML framework with parameterless constructors
- Current pattern: `App.Current.Services.GetService<T>()` in constructor body

**Migration Strategy:**
1. **Add DI constructors** alongside parameterless (no breaking changes)
2. **Create IPageFactory** service for DI-aware page creation
3. **Update navigation** to use factory instead of `Frame.Navigate(typeof(Page))`
4. **Migrate MainWindow** in `App.OnLaunched` to inject dependencies
5. **Test incrementally** per subsystem
6. **Remove service locator** once all call sites migrated

**Risk Assessment:** **MODERATE**
- No circular deps = low risk
- XAML instantiation = moderate complexity (factory pattern)
- Estimated effort: 2-3 weeks with testing

**Key File Paths:**
- Service registrations: `src/modules/cmdpal/Microsoft.CmdPal.UI/App.xaml.cs`
- Service implementations: `Microsoft.CmdPal.UI.ViewModels/Services/`, `Microsoft.CmdPal.UI/Services/`, `Microsoft.CmdPal.Common/Services/`
- Pages using service locator: `src/modules/cmdpal/Microsoft.CmdPal.UI/Settings/*.xaml.cs`, `MainWindow.xaml.cs`
- Full analysis: `.squad/agents/hawk/di-analysis.md`
- Rich navigation model (Stack-based with HomeGoBack/GoHome variants) supports complex UIs
- Telemetry + logging infrastructure (PowerToys.Telemetry, spdlog) for monitoring

**Concerns & Areas for Attention:**
1. **Singleton Anti-pattern**: `CommandPaletteHost.Instance` is a static singleton; should migrate to full DI
2. **COM Activation Risk**: In-proc extension loading can crash host if extension has bug (no isolation)
3. **Thread Safety**: `TaskScheduler` marshalling implicit in many places; could add thread-safe wrappers
4. **Mixed Architecture**: C++ module interface + C# UI + WinRT IDL extensions adds complexity for contributors
5. **Settings Migration**: Changing `ProviderSettings` schema requires careful migration (no schema versioning visible)
6. **Extension Unload**: No visible unload/cleanup path for extensions; long-lived process risk
7. **Keyboard Hook STA Requirement**: Program.cs comment says Main MUST be STA (no MTA), clipboard dependency

**File Paths of Interest:**
- Extension SDK: `src/modules/cmdpal/extensionsdk/Microsoft.CommandPalette.Extensions/Microsoft.CommandPalette.Extensions.idl`
- Toolkit: `src/modules/cmdpal/extensionsdk/Microsoft.CommandPalette.Extensions.Toolkit/` (C# wrapper)
- Module Interface: `src/modules/cmdpal/CmdPalModuleInterface/dllmain.cpp` (PowerToys runner entry)
- Extension Host: `src/modules/cmdpal/Microsoft.CmdPal.UI.ViewModels/AppExtensionHost.cs`, `CommandPaletteHost.cs`
- Extension Loading: `src/modules/cmdpal/Microsoft.CmdPal.UI.ViewModels/CommandProviderWrapper.cs`
- Main App: `src/modules/cmdpal/Microsoft.CmdPal.UI/Program.cs`, `MainWindow.xaml.cs`
- Tests: 16 projects under `src/modules/cmdpal/Tests/` (unit + UI tests with WinAppDriver)
- Built-in Extensions: `src/modules/cmdpal/ext/` (Microsoft.CmdPal.Ext.Apps, .Calc, .Shell, etc.)

### 2026-04-21: Code Review - Pluralization Fix (Issue #47110)

**Branch:** `dev/mjolley/fix-cmdpal-plural-form`  
**Issue:** #47110 — CmdPal uses plural form in settings ("1 commands" instead of "1 command")

**Files Reviewed:**
1. `src/modules/cmdpal/Microsoft.CmdPal.UI.ViewModels/ProviderSettingsViewModel.cs`
2. `src/modules/cmdpal/Microsoft.CmdPal.UI.ViewModels/Properties/Resources.resx`
3. `src/modules/cmdpal/Tests/Microsoft.CmdPal.UI.ViewModels.UnitTests/ProviderSettingsViewModelPluralizationTests.cs`

**Findings:**

✅ **Code correctness:** Tuple pattern matching covers all 4 combinations (singular/plural × command/fallback). Logic is sound.

✅ **Edge case (count=0):** Uses plural form ("0 commands"), which is linguistically correct in English.

✅ **Resource naming:** Consistent with existing conventions (`builtin_extension_subtext_*` pattern). Comments clearly indicate singular/plural cases.

✅ **Pattern consistency:** Matches existing pluralization pattern in `DockBandSettingsViewModel.cs` (line 35-37), which uses ternary for singular/plural selection with separate resource strings.

✅ **StyleCop/editorconfig compliance:** Indentation (4 spaces), formatting, naming conventions all correct per `src/.editorconfig`.

✅ **Test coverage:** Comprehensive test suite covers all cases: 0 commands, 1 command, 2+ commands, fallback variants, disabled state, built-in providers. Test infrastructure matches existing patterns in other `*Tests.cs` files (Mock usage, helper methods, MSTest attributes).

**Unrelated change noted:**
- `Microsoft.CmdPal.UI.csproj` adds `<EventSourceSupport>true</EventSourceSupport>` — outside scope of pluralization fix, but appears to be for AOT build telemetry. Not a concern.

**Pattern Analysis:**
- CmdPal uses CompositeFormat with separate resource strings for singular/plural (not conditional formatting within a single string)
- This is consistent across the codebase: DockBandSettingsViewModel, SettingsExtensionsViewModel both follow this pattern
- No other pluralization bugs found in ViewModels layer

**Verdict:** ✅ APPROVE — Code is correct, follows existing patterns, comprehensive tests, no style violations.

### 2026-04-21: Issue #46634 Analysis — Hide app descriptions in Command Palette

**Issue:** #46634 — CmdPal: Hide descriptions for listed applications  
**Status:** Analysis only (no code changes)

**Investigation Summary:**

✅ **List Item Subtitle Field:**
- Defined in Extension SDK IDL: `IListItem` requires `ICommandItem`, which has a `String Subtitle { get; }` property
- Subtitles populated in `AppListItem` ctor from `AppItem.Subtitle` (line 94 of AppListItem.cs)
- UWP apps: Description from manifest (UWPApplication.ToAppItem, line 364)
- Win32 apps: Description from FileVersionInfo (Win32Program.ToAppItem, line 1086)
- Extensions can freely set Subtitle to empty string; UI gracefully renders nothing

✅ **Settings Infrastructure:**
- AllAppsSettings extends `JsonSettingsManager` (from Extension SDK Toolkit)
- Settings stored in JSON at `%APPDATA%/Microsoft.CmdPal/apps.settings.json`
- Pattern uses `ToggleSetting` (boolean) or `ChoiceSetSetting` (dropdown)
- Each setting auto-generates UI card (Adaptive Cards) — no manual UI work needed
- Settings are namespaced: `Namespaced(nameof(PropertyName))` → `apps.{PropertyName}`

✅ **Existing Toggle Patterns:**
- `EnableStartMenuSource`, `EnableDesktopSource`: Both use ToggleSetting pattern
- Resource strings follow naming convention: `enable_start_menu_source` (label), description
- Settings constructor registers each setting: `Settings.Add(_field)`

✅ **Fallback Behavior:**
- Mentioned in issue ("app list and fallback") refers to SearchResultLimit setting
- Already applies uniformly to both main results and fallback set
- No special handling needed for new hide-descriptions setting

**Proposed Solution:**
1. Add `ToggleSetting _hideAppDescriptions` field to AllAppsSettings (default: false)
2. Add public `HideAppDescriptions` property
3. Add resource strings: `hide_app_descriptions`, `hide_app_descriptions_description`
4. In AllAppsPage.BuildListItems() or GetPrograms(): conditionally clear Subtitle if setting enabled
5. Unit tests: verify setting persists and toggles correctly

**Files to Modify:**
1. `src/modules/cmdpal/ext/Microsoft.CmdPal.Ext.Apps/AllAppsSettings.cs` — add ToggleSetting field + property + register
2. `src/modules/cmdpal/ext/Microsoft.CmdPal.Ext.Apps/Properties/Resources.resx` — add 2 strings
3. `src/modules/cmdpal/ext/Microsoft.CmdPal.Ext.Apps/AllAppsPage.cs` — conditionally clear subtitle in BuildListItems()
4. Unit tests (if applicable): `AllAppsSettingsTests.cs` or similar

**Risks:** None. Default false = backward compatible. No migration needed (JsonSettingsManager handles new settings gracefully). Negligible perf impact (one bool check during list build).

**Pattern Confidence:** ✅ High. Mirrors existing `EnableStartMenuSource` pattern exactly.

### 2026-04-21: Code Review — Tag Overflow Fix (Issue #38317)

**Branch:** `agents/issue-38317-preparation`
**Issue:** #38317 — Too many tags on a list item pushes/truncates title text

**Files Reviewed:**
1. `src/modules/cmdpal/Microsoft.CmdPal.UI.ViewModels/ListItemViewModel.cs` — ViewModel capping logic
2. `src/modules/cmdpal/Microsoft.CmdPal.UI/ExtViews/ListPage.xaml` — XAML template with overflow badge

**Key Findings:**

✅ **Contract stability:** Original `Tags` and `HasTags` properties preserved. New additive properties (`VisibleTags`, `OverflowTagCount`, `HasOverflowTags`, `OverflowTagText`). `DetailsTagsViewModel` on ShellPage is a completely separate class — unaffected.

✅ **MVVM separation:** Capping logic lives entirely in `ListItemViewModel.UpdateVisibleTags()`. No code-behind. XAML binds to computed properties only.

✅ **Theme resource reuse:** Overflow badge correctly reuses `TagPadding`, `TagBackground`, `TagBorderBrush`, `TagBorderThickness`, `TagForeground`, `ControlCornerRadius` from Tag.xaml resource dictionary. `FontSize="12"` matches TagTemplate's Tag control.

✅ **Edge cases handled:** null→no-op, 0→hidden, 1-3→no badge, 4+→first 3 + "+N" badge.

✅ **Accessibility:** `AutomationProperties.Name` set on overflow border. Noted improvement opportunity: "+3" is less descriptive than "+3 more tags" for screen readers.

✅ **Performance:** `Take(3).ToList()` is O(3) constant. Called only on tag change events, not hot paths.

**Layout note:** The outer Grid uses `Width="*"` for title, `Width="Auto"` for tags. WinUI measures Auto first, so tags technically get layout priority. With the cap at 3, the Auto column is bounded and the title gets adequate space. True layout priority inversion would require column definition changes — higher risk, lower reward given the cap.

**Verdict:** 🔄 APPROVED WITH SUGGESTIONS — ship it, follow up on accessible name improvement.

**Patterns Documented:**
- Tag overflow pattern: ViewModel caps + derived properties, XAML binds `VisibleTags` + conditional overflow badge
- ListPage.xaml Grid layout: Col 0=28px icon, Col 1=* title/subtitle, Col 2=Auto tags
- Tag theme resources defined in `Controls/Tag.xaml` — reusable for any tag-like element

### Full DI Investigation Summary (2026-04-22)

**Task:** Complete dependency graph analysis across 30+ services, 5 dependency levels, identification of all components requiring migration.

**Key Findings:**
- ✅ **Zero circular dependencies** confirmed across entire codebase
- ✅ **13 XAML pages + 4 controls** identified for DI migration
- ✅ **5 dependency levels** mapped cleanly from UI through to shared libraries
- ✅ **No architectural blockers** — DI migration is feasible

**Dependency Hierarchy (Clean Layering):**
1. **UI Layer** (XAML pages/controls) — depends on ViewModels + services
2. **ViewModel Layer** — depends on services + managers
3. **Service Layer** — depends on persistence + utilities
4. **Shared Services** — no circular dependencies
5. **Utilities & Extensions** — leaf nodes

**Cross-Agent Collaboration:**
- **Scarlett (UI Layer Audit):** Found 12 files with 47 service calls. ISettingsService most coupled (24 calls). Property-injection pattern required for XAML.
- **Snake Eyes (ViewModel/Service Analysis):** Confirmed zero circular deps, identified 2 quick wins (HotkeyManager injection, DefaultContextMenuFactory DI).
- **This Investigation:** Full dependency graph with no blockers for proceeding.

**Status:** Architecture is clean and DI-ready. No blocking issues. Ready for 3-phase implementation strategy.
