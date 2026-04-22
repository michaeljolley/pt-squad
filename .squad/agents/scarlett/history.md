# Project Context

- **Owner:** Michael Jolley
- **Project:** Microsoft Command Palette (CmdPal) — a PowerToys utility providing a command launcher with extensible plugin architecture
- **Stack:** C# (WinUI 3 / WinAppSDK), C++/WinRT, XAML, WinRT IDL
- **Scope:** `src/modules/cmdpal/` and `CommandPalette.slnf` only
- **Created:** 2026-04-20

## Learnings

### UI Architecture & Structure

**Project Layout:** The UI is organized into three main projects:
- `Microsoft.CmdPal.UI` (XAML/C# views and code-behind)
- `Microsoft.CmdPal.UI.ViewModels` (MVVM ViewModels and business logic)
- `Microsoft.CmdPal.Common` (shared utilities, logging, services)

**Key Directories:**
- `Pages/` — Root pages (ShellPage.xaml is main content area)
- `Controls/` — Custom XAML controls (SearchBar, CommandBar, Tag, IconBox, BlurImageControl, CommandPalettePreview, FiltersDropDown, etc.)
- `Styles/` — Theme dictionaries (Theme.Normal.xaml, Theme.Colorful.xaml, TextBlock.xaml, TextBox.xaml, Settings.xaml)
- `Converters/` — Custom ValueConverters (ListItemTemplateSelector, DetailsDataTemplateSelector, KeyChordToStringConverter, DetailsSizeToGridLengthConverter, etc.)
- `Services/` — UI services (ThemeService, ResourceSwapper, WindowThemeSynchronizer, TrayIconService)
- `ExtViews/` — Extension-specific views (ListPage, ContentPage with specialized content viewers for plain text, images, markdown)
- `Dock/` — Dock window and controls (DockWindow.xaml, DockControl.xaml, DockItemControl.xaml, DockContentControl.xaml)
- `Settings/` — Settings UI (SettingsWindow.xaml with NavigationView pattern, pages for General, Appearance, Extensions, Dock settings)

### MVVM & Data Binding Patterns

**ViewModel Base:** Uses Community Toolkit MVVM (`CommunityToolkit.Mvvm`):
- `ObservableObject` base class for automatic INotifyPropertyChanged
- `[ObservableProperty]` code generator for public properties
- `[NotifyPropertyChangedFor(nameof(...))]` for dependent property notifications
- `PageViewModel` is base for pages (abstract, has IsLoaded, IsInitialized, PlaceholderText, ErrorMessage, SearchTextBox, StatusMessage)
- `ExtensionObjectViewModel` is base for extension-aware VMs (handles IBatchUpdateTarget, background thread PropChanged events, weak ref to IPageContext)

**Data Binding:**
- Uses x:Bind (compiled binding) exclusively — Mode="OneWay" for most properties
- Binding to extension model objects wrapped in `ExtensionObject<T>` (handles COM proxy state)
- ViewModels access model properties via `Model.Unsafe` (com proxy) with null checks
- PropChanged events from extension objects arrive on background thread and must be marshaled to UI thread via `UpdateProperty()` in batch updates

**Key ViewModels:**
- `ShellViewModel` — Manages CurrentPage navigation, Details visibility, message routing (IRecipient pattern)
- `ListViewModel` — Handles paginated list items with incremental refresh, filtering, empty content state
- `MainWindowViewModel` — Manages background image, backdrop effects, theme bindings
- `DetailsViewModel` — Shows details panel with metadata (tags, links, separators, commands)
- `CommandViewModel` — Represents a single command with Id, Name, Icon properties
- `DockViewModel` — Manages dock bands in three sections (Start, Center, End)

### Theme & Styling

**Theme Service (`Services/ThemeService.cs`):**
- Observes Windows system theme changes (UISettings listener)
- Reads settings (ColorizationMode, CustomThemeColor, BackgroundImagePath, BackdropStyle)
- Determines theme provider: `NormalThemeProvider` or `ColorfulThemeProvider` based on intensity
- Updates `ResourceSwapper` with computed colors and brushes (Mutable overrides dictionary)
- Emits `ThemeChanged` event with full snapshot

**Resource Management:**
- `ResourceSwapper` merges dynamic theme into application resources at `MutableOverridesDictionary`
- `App.xaml` includes theme dictionaries in mergedDictionaries:
  - TeachingTip.xaml, TextBlock.xaml, TextBox.xaml, Settings.xaml, Tag.xaml (custom)
  - Theme.Normal.xaml (default with Dark/Light/HighContrast theme dicts)
- Custom WinUI theme brushes: `LayerOnAcrylicPrimaryBackgroundBrush`, `LayerOnAcrylicSecondaryBackgroundBrush`, `CmdPal.TopBarBorderBrush`, etc.
- HighContrast support via theme dictionaries using SystemColor values

**Custom Controls & Styles:**
- `Tag.xaml` — Custom tag control with icon, background, border, theme support
- `TextBlock.xaml`, `TextBox.xaml` — Style overrides (caret color sync, selection)
- `Settings.xaml` — Settings-specific style definitions

### Navigation & Messaging

**Navigation:**
- Uses `NavigateToPageMessage` record (page, withAnimation, cancellationToken, transientPage)
- ShellPage registered as IRecipient for multiple message types: NavigateBackMessage, NavigateToPageMessage, ShowDetailsMessage, etc.
- Frame-based navigation with slide transition support (SlideNavigationTransitionInfo)
- Back/forward stack managed implicitly

**Message Passing (Weak Reference Messenger pattern):**
- Pages/ViewModels register with `WeakReferenceMessenger.Default.Register<TMessage>(this)`
- Loose coupling via record-based messages in `Microsoft.CmdPal.UI.ViewModels.Messages/`
- Examples: `PerformCommandMessage`, `HandleCommandResultMessage`, `WindowHiddenMessage`, `ShowToastMessage`, `ShowDetailsMessage`, `DismissMessage`
- Messaging handled on UI dispatcher via `DispatcherQueue`

### Accessibility

**AutomationProperties:**
- SearchBar (FilterBox) — `AutomationProperties.AutomationId="MainSearchBox"`
- Tag control template — `AutomationProperties.Name="{TemplateBinding Text}"`
- Icon image — `AutomationProperties.AccessibilityView="Raw"` (decorative)
- Custom Tag — `AutomationProperties.AutomationControlType="Custom"`
- Limited implementation — not comprehensive across all controls; opportunity for improvement

**Details:** Count of AutomationProperties usage shows 12 XAML files with automation support, but coverage is selective rather than systematic.

### Data Templates & Type Selection

**Template Selection:**
- `ListItemTemplateSelector` — Selects template based on ListItemType (Item, Separator, SectionHeader)
- `DetailsDataTemplateSelector` — Maps details metadata types (Commands, Links, Separators, Tags)
- `GridItemTemplateSelector`, `GridItemContainerStyleSelector` — For grid view layouts
- `FilterTemplateSelector`, `ContextItemTemplateSelector`, `ContentTemplateSelector` — Domain-specific selectors
- `ListItemContainerStyleSelector` — Applies different container styles per item type

### Custom Controls

**High-Impact Custom Controls:**
- `BlurImageControl` — Advanced image compositing with blur, brightness, tint, using Composition effects
- `IconBox` — Generic icon renderer; accepts IconSource or SourceKey with external resolution
- `ScrollContainer` — Custom scroll handling
- `ScreenPreview` — Renders screen preview for background image settings
- `CommandPalettePreview` — Shows backdrop style preview (Clear, Acrylic, Mica with tint/opacity)
- `ContextMenu` — Custom context menu
- `FiltersDropDown` — Filterable dropdown
- `SearchBar` — Search textbox with key event handling (KeyDown, PreviewKeyDown, PreviewKeyUp)
- `ShortcutControl` — Keyboard shortcut capture with native hook support (NativeKeyboardHelper, HotkeySettingsControlHook)

### Settings & Persistence

**SettingsService (`Services/SettingsService.cs`):**
- Loads/saves `SettingsModel` to JSON file via `IPersistenceService`
- Handles migrations on load (parses old JSON schema, updates to new)
- Thread-safe using `Volatile` reads/writes and `Interlocked.CompareExchange`
- Emits `SettingsChanged` event after save (with optional hotReload flag)
- Settings include: CustomThemeColor, ColorizationMode, BackdropStyle, BackdropOpacity, BackgroundImage settings, HotkeySummary, DockSettings, ProviderSettings

**Settings UI:**
- `SettingsWindow.xaml` uses NavigationView with pages:
  - GeneralPage, AppearancePage, ExtensionsPage, ExtensionPage, DockSettingsPage, InternalPage
- Settings persisted via SettingsService after UI changes
- Dock settings stored in separate DockSettings object

### Extension Object Handling

**ExtensionObjectViewModel:**
- Wraps extension model objects (IPage, ICommand, IDetails, etc.) in `ExtensionObject<T>`
- Monitors PropChanged events from COM proxy objects arriving on background thread
- Implements `IBatchUpdateTarget` for efficient batch updates (queues pending property names, applies in batches)
- Implements `IBackgroundPropertyChangedNotification` to intercept background thread notifications
- Uses weak references to IPageContext to avoid circular references

**Extension Models:**
- Modeled after PowerToys Extensions API (Microsoft.CommandPalette.Extensions)
- Each extension type has corresponding ViewModel wrapper for binding

### Common Library (`Microsoft.CmdPal.Common`)

**Key Files:**
- `CoreLogger.cs` — Logging façade (delegates to PT logging)
- `Services/` — Base interfaces (ISettingsService, IPersistenceService, IApplicationInfoService)
- `Helpers/` — Utility code for logging, JSON, text processing
- `Messages/` — Shared message types

### Strengths

1. **Clean MVVM separation** — ViewModels are independent of UI framework (testable)
2. **Efficient data binding** — Uses x:Bind (compiled) with OneWay bindings
3. **Theme infrastructure** — Comprehensive system for dark/light/high contrast with resource swapping
4. **Custom controls** — Well-crafted specialized controls (BlurImageControl, IconBox, Tag)
5. **Message-driven architecture** — Loose coupling via weak reference messenger
6. **Extension integration** — Clean wrapper pattern for extension objects with background thread safety
7. **Async/cancellation support** — Navigation supports CancellationToken, proper async patterns

### Service Locator Anti-Pattern (2026-04-21)

**Context:** Michael requested audit of all `App.Current.Services` usage in UI layer to enable migration to constructor injection.

**Files with Service Locator (12 total):**
- `MainWindow.xaml.cs` (9 calls) — Root window; ISettingsService, IThemeService, TrayIconService, IExtensionService, MainWindowViewModel
- `ShellPage.xaml.cs` (6 calls) — Main content page; ShellViewModel, ISettingsService, TopLevelCommandManager
- `ListPage.xaml.cs` (2 calls) — Extension list view; ISettingsService (event handlers)
- `DockWindow.xaml.cs` (1 call, multi-resolution) — Dock window; ISettingsService, DockViewModel, IThemeService
- `ContextMenu.xaml.cs` (1 call) — Context menu control; IFuzzyMatcherProvider (for ViewModel)
- `FallbackRanker.xaml.cs` (3 calls) — Settings control; TopLevelCommandManager, IThemeService, ISettingsService (for ViewModel)
- `SearchBar.xaml.cs` (1 call) — Search control; ISettingsService (property getter)
- `AppearancePage.xaml.cs` (3 calls) — Settings page; IThemeService, TopLevelCommandManager, ISettingsService (for ViewModel)
- `DockSettingsPage.xaml.cs` (8 calls) — Settings page; IThemeService, TopLevelCommandManager, ISettingsService, DockViewModel
- `ExtensionsPage.xaml.cs` (3 calls) — Settings page; TopLevelCommandManager, IThemeService, ISettingsService (for ViewModel)
- `GeneralPage.xaml.cs` (4 calls) — Settings page; TopLevelCommandManager, IThemeService, ISettingsService, IApplicationInfoService
- `InternalPage.xaml.cs` (1 call) — Settings page; IApplicationInfoService

**Service Usage Frequency:**
- `ISettingsService` — 24 calls across 10 files (most frequently resolved)
- `TopLevelCommandManager` — 9 calls across 7 files
- `IThemeService` — 7 calls across 7 files
- ViewModels (MainWindowViewModel, ShellViewModel, DockViewModel) — 5 calls
- `TrayIconService` — 3 calls (MainWindow only)
- `IApplicationInfoService` — 2 calls
- `IFuzzyMatcherProvider`, `IExtensionService` — 1 call each

**Patterns Discovered:**
1. **Caching Service Provider** — `MainWindow.xaml.cs:904`, `DockWindow.xaml.cs:75` store `App.Current.Services` in local variable
2. **Event Handler Resolutions** — 5 occurrences where services resolved inside frequently-called event handlers (mouse clicks, message handlers)
3. **Property Getter Resolutions** — `ShellPage.DefaultPageAnimation` and `SearchBar.Settings` properties resolve services on every access
4. **Transitive Resolutions** — 6 files resolve services only to pass to ViewModel constructors (ContextMenu, FallbackRanker, settings pages)

**XAML Instantiation Challenge:**
All pages, controls, and windows are XAML-instantiated via parameterless constructors. WinUI framework creates these via XAML parser, preventing traditional constructor injection. Migration requires:
1. Property injection pattern (set after construction via App or factory)
2. Factory pattern for code-created types (DockWindow)
3. ViewModel-based DI (inject ViewModel instead of creating in code-behind)
4. Navigation parameter-based injection (pass services via Frame.Navigate parameter)

**Migration Difficulty Ratings:**
- **HARD (2 files):** MainWindow, ShellPage — Complex lifecycle, multiple services, event subscriptions, framework-managed
- **MEDIUM (4 files):** ListPage, DockWindow, ContextMenu, FallbackRanker, DockSettingsPage — Event handlers or code-created with call site dependencies
- **EASY (6 files):** SearchBar, AppearancePage, ExtensionsPage, GeneralPage, InternalPage — Single service or ViewModel-only usage

**Recommended Migration Strategy:**
- **Phase 1 (Quick Wins):** SearchBar, AppearancePage, ExtensionsPage, GeneralPage, InternalPage — Property-inject services or ViewModels
- **Phase 2 (ViewModel Refactoring):** ContextMenu, FallbackRanker, DockWindow, ListPage, DockSettingsPage — Inject ViewModels externally, move queries to ViewModels
- **Phase 3 (Core Infrastructure):** MainWindow, ShellPage — Property-inject core services, careful coordination with App lifecycle

**Key Risk:** XAML instantiation timing. Services injected via properties must be set BEFORE the control/page is used. Requires coordination between App/Factory and XAML creation to avoid `NullReferenceException`.

**Detailed Audit:** See `~/.copilot/session-state/service-locator-audit.md` for full per-file breakdown with line numbers and migration recommendations.

### Concerns & Improvement Opportunities

1. **Accessibility Coverage** — AutomationProperties are sparse; many controls lack Names/Descriptions for screen readers
2. **Code-behind Complexity** — ShellPage.xaml.cs is large with multiple message handlers; could benefit from behaviors/attached behaviors
3. **Custom Theme Selectors** — Template selectors are simple but could use consolidation (many similar patterns)
4. **Hardcoded Values** — Some magic numbers in styles (TagPadding: 4,2,4,2; TagBorderThickness: 1)
5. **Details Panel** — Switching details visibility with animation might have timing edge cases
6. **Dock Implementation** — DockViewModel logic is complex; EditMode state transitions need careful handling
7. **Error Handling** — GlobalErrorHandler exists but pattern is inconsistent across VMs
8. **Testing Gaps** — UI is not directly testable (WinUI + platform specific); ViewModels are testable but integration needs work

### Pluralization Patterns (2026-04-21)

**Issue #47110 - Extensions settings plural/singular forms**

**Problem:** Extensions page showed "1 commands" and "1 fallback commands" instead of using singular forms when count is 1.

**Solution Pattern:**
- Create multiple resource strings in `Properties/Resources.resx` for each singular/plural combination
- Use pattern switch expression in ViewModel to select the correct format based on count
- For dual counts (commands + fallback commands), use tuple pattern matching: `(commandSingular, fallbackSingular) switch`

**Key Files:**
- `Microsoft.CmdPal.UI.ViewModels\Properties\Resources.resx` — Resource strings for localization
- `Microsoft.CmdPal.UI.ViewModels\ProviderSettingsViewModel.cs` — Extension settings ViewModel that formats the subtext

**Pattern Example:**
```csharp
// Create static CompositeFormat instances for each variant
private static readonly CompositeFormat FormatSingular = CompositeFormat.Parse(Resources.resource_singular);
private static readonly CompositeFormat FormatPlural = CompositeFormat.Parse(Resources.resource_plural);

// Select format based on count
CompositeFormat format = count == 1 ? FormatSingular : FormatPlural;
return string.Format(CultureInfo.CurrentCulture, format, name, count);
```

**Dual-count pattern matching:**
```csharp
bool commandSingular = commandCount == 1;
bool fallbackSingular = fallbackCount == 1;

CompositeFormat format = (commandSingular, fallbackSingular) switch
{
    (true, true) => BothSingularFormat,
    (true, false) => CommandSingularFormat,
    (false, true) => FallbackSingularFormat,
    (false, false) => BothPluralFormat,
};
```

**Decision:** CmdPal does not use a pluralization library or framework. Each pluralization case requires explicit resource strings and conditional logic. This ensures full control over localization and avoids runtime dependencies.

### UI Layer DI Audit Summary (2026-04-22)

**Task:** Comprehensive audit of `App.Current.Services` usage across the entire UI layer to identify service locator calls and migration strategy.

**Audit Results:**
- **12 files** using `App.Current.Services` to resolve services
- **47 service resolution calls** identified
- **ISettingsService** most heavily coupled (24 calls across 10 files)
- **TopLevelCommandManager** second most common (9 calls across 7 files)

**Anti-Patterns Found:**
1. **Caching Service Provider** — 2 files store `App.Current.Services` in local variable
2. **Event Handler Resolutions** — 5 occurrences resolve services in frequently-called event handlers
3. **Property Getter Resolutions** — 2 occurrences resolve services on every property access
4. **Transitive Resolutions** — 6 files resolve services only to pass to ViewModels

**Difficulty Classification:**
- **EASY (6 files):** SearchBar, AppearancePage, ExtensionsPage, GeneralPage, InternalPage + 1 more
- **MEDIUM (5 files):** ListPage, DockWindow, ContextMenu, FallbackRanker, DockSettingsPage
- **HARD (2 files):** MainWindow, ShellPage (complex lifecycle, framework-managed constructors)

**Critical Challenge:** XAML pages/controls instantiated via parameterless constructors by framework. Cannot use traditional constructor injection. Solution: property injection with careful App/factory coordination.

**Three-Phase Migration Strategy:**
1. **Phase 1 (EASY):** Property-inject services/ViewModels into 6 files (~12 calls removed)
2. **Phase 2 (MEDIUM):** Refactor ViewModels, inject externally into 5 files (~17 calls removed)
3. **Phase 3 (HARD):** Property-inject core services with careful App lifecycle coordination (~15 calls removed)

**Risk Mitigation:** Services set via properties MUST be available BEFORE control/page usage. Requires tight coordination between App and factories. Recommend prototyping Phase 1 first to validate pattern.
