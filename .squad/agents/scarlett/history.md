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

### Concerns & Improvement Opportunities

1. **Accessibility Coverage** — AutomationProperties are sparse; many controls lack Names/Descriptions for screen readers
2. **Code-behind Complexity** — ShellPage.xaml.cs is large with multiple message handlers; could benefit from behaviors/attached behaviors
3. **Custom Theme Selectors** — Template selectors are simple but could use consolidation (many similar patterns)
4. **Hardcoded Values** — Some magic numbers in styles (TagPadding: 4,2,4,2; TagBorderThickness: 1)
5. **Details Panel** — Switching details visibility with animation might have timing edge cases
6. **Dock Implementation** — DockViewModel logic is complex; EditMode state transitions need careful handling
7. **Error Handling** — GlobalErrorHandler exists but pattern is inconsistent across VMs
8. **Testing Gaps** — UI is not directly testable (WinUI + platform specific); ViewModels are testable but integration needs work
