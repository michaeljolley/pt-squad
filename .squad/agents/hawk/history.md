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
