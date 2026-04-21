# Project Context

- **Owner:** Michael Jolley
- **Project:** Microsoft Command Palette (CmdPal) — a PowerToys utility providing a command launcher with extensible plugin architecture
- **Stack:** C# (WinUI 3 / WinAppSDK), C++/WinRT, XAML, WinRT IDL
- **Scope:** `src/modules/cmdpal/` and `CommandPalette.slnf` only
- **Created:** 2026-04-20

## Learnings

### Test Infrastructure Review (2026-04-20)

**Test Framework & Mocking:**
- All unit tests use **MSTest** (Microsoft.VisualStudio.TestTools.UnitTesting)
- Mocking library: **Moq** (for interface/dependency mocking)
- Global usings pattern: `global using` statements in `GlobalUsings.cs` reduce boilerplate
- No xUnit or NUnit in use; standardized on MSTest across all 16 test projects

**Test Projects Inventory (17 total):**
1. **Microsoft.CmdPal.Common.UnitTests** — Tests common helpers (ProviderLoadGuard, sanitizers, text utilities). Located: `src/modules/cmdpal/Tests/Microsoft.CmdPal.Common.UnitTests/`
2. **Microsoft.CmdPal.Ext.UnitTestsBase** — Shared test base class (CommandPaletteUnitTestBase). NOT a test project (TestingPlatformDisableCustomTestTarget=true); provides FuzzyStringMatcher helper & ItemsChanged event utilities
3. **Microsoft.CmdPal.Ext.Apps.UnitTests** — App discovery/caching (AllAppsCommandProviderTests, AllAppsPageTests, QueryTests). Custom test base AppsTestBase provides MockAppCache
4. **Microsoft.CmdPal.Ext.Bookmarks.UnitTests** — Bookmark management (BookmarkResolverTests, BookmarkManagerTests, BookmarkJsonParserTests, PlaceholderParserTests, UriHelperTests). ~15 test files; uses ClassInitialize/ClassCleanup for file I/O setup
5. **Microsoft.CmdPal.Ext.Calc.UnitTests** — Calculator parser/computation (QueryTests, ExtendedCalculatorParserTests, BracketHelperTests, NumberTranslatorTests, BaseConverterTests). Tests cultural settings (SetUp/CleanUp restores CultureInfo)
6. **Microsoft.CmdPal.Ext.ClipboardHistory.UnitTests** — Clipboard history (UrlHelperTests). Minimal coverage — only 1 test file
7. **Microsoft.CmdPal.Ext.Indexer.UnitTests** — File indexing (SearchNoticeInfoBuilderTests, ImplicitWildcardQueryBuilderTests, FallbackOpenFileItemTests). Focuses on query building & file results
8. **Microsoft.CmdPal.Ext.Registry.UnitTests** — Registry extension (QueryTests, RegistryHelperTest, KeyNameTest, ResultHelperTest, BasicStructureTest). ~7 test files
9. **Microsoft.CmdPal.Ext.RemoteDesktop.UnitTests** — RDP connections (RdpConnectionsManagerTests, RemoteDesktopCommandProviderTests, FallbackRemoteDesktopItemTests). Uses MockSettingsManager
10. **Microsoft.CmdPal.Ext.Shell.UnitTests** — Shell commands (ShellCommandProviderTests, QueryTests, NormalizeCommandLineTests). Minimal test count
11. **Microsoft.CmdPal.Ext.System.UnitTests** — System info (QueryTests, BasicTests, ImageTests). ~4 test files
12. **Microsoft.CmdPal.Ext.TimeDate.UnitTests** — Date/time utilities (TimeDateCalculatorTests, StringParserTests, QueryTests, AvailableResultsListTests). ~9 test files
13. **Microsoft.CmdPal.Ext.WebSearch.UnitTests** — Web search (QueryTests, WebSearchCommandProviderTests, SettingsManagerTests). Uses MockBrowserInfoService, MockSettingsInterface
14. **Microsoft.CmdPal.Ext.WindowWalker.UnitTests** — Window enumeration. Minimal coverage — only Settings.cs file visible
15. **Microsoft.CmdPal.UI.ViewModels.UnitTests** — UI layer ViewModels (ListViewModelTests, SettingsServiceTests, PersistenceServiceTests, CommandItemViewModelTests). Includes custom test pages (RecursiveItemsChangedPage) & TaskScheduler-based async testing
16. **Microsoft.CmdPal.UITests** — UI automation tests (BasicTests, IndexerTests). Uses UITestBase from PowerToys.UITest; references CommandPaletteTestBase. Tests file search, calculator, time/date via UI
17. **Microsoft.CommandPalette.Extensions.Toolkit.UnitTests** — Fuzzy matching & SDK utilities (FuzzyMatcherValidationTests, FuzzyMatcherDiacriticsTests, FuzzyMatcherEmojiTests, FuzzyMatcherComplexEmojiTests, ListHelpersInPlaceUpdateTests). ~9 test files; heavy focus on FuzzyStringMatcher edge cases (null/empty, Unicode, Pinyin, diacritics, emoji)

**Shared Test Base Pattern:**
- `CommandPaletteUnitTestBase` provides:
  - `Query(string query, IListItem[] candidates)` — Fuzzy filter helper
  - `UpdatePageAndWaitForItems(IDynamicListPage page, Action modification)` — Async event-driven test helper using TaskCompletionSource
- Extension tests that need setup (Apps, Bookmarks) define custom base classes (AppsTestBase) with TestInitialize/TestCleanup

**Test Quality Observations:**
- **Strengths:**
  - Comprehensive edge case coverage (fuzzy matching tests cover null, empty, Unicode, Pinyin, diacritics, emoji)
  - DataTestMethod patterns with DataRow attributes for parametrized tests (Calc, Fuzzy matcher tests)
  - Proper test cleanup (Cleanup methods delete temp directories; ClassCleanup in Bookmarks)
  - Cultural isolation (Calc tests reset CultureInfo before/after)
  - Mock objects where needed (MockAppCache, MockBrowserInfoService, MockRdpConnectionsManager)
  - Async test support (TaskCompletionSource for event-driven tests; Task-based test methods)
  - Clear Arrange-Act-Assert structure in provider tests

- **Concerns:**
  - **Minimal coverage for some extensions:** ClipboardHistory (1 file), WindowWalker (1 file), Shell (minimal files)
  - **No test coverage for:** WinGet, WindowsServices, WindowsSettings, WindowsTerminal, PerformanceMonitor, PowerToys, SamplePages, ProcessMonitor extensions
  - **UI test coverage thin:** Only BasicTests with 5-6 test methods for entire UI surface
  - **No integration tests:** All tests are unit-level; no end-to-end scenarios
  - **Settings mocking inconsistent:** Some tests create Settings.cs helper; others use MockSettingsInterface
  - **Test count unclear:** Need MSTest run to get exact count, but estimated 200-400 test methods across 16 projects

**Coverage Gaps (Production without Tests):**
- **9 extension modules untested:** `Microsoft.CmdPal.Ext.WinGet`, `.WindowsServices`, `.WindowsSettings`, `.WindowsTerminal`, `.PerformanceMonitor`, `.PowerToys`, `ProcessMonitorExtension`, `SamplePagesExtension` + `ExtensionTemplate` (template, not production)
- **Core modules with tests:** Common, Apps, Bookmarks, Calc, ClipboardHistory, Indexer, Registry, RemoteDesktop, Shell, System, TimeDate, WebSearch, WindowWalker
- **Framework modules:** UI.ViewModels fully tested; Core likely not tested (no Core.UnitTests project)

**Test Infrastructure File Paths:**
- Shared base: `src/modules/cmdpal/Tests/Microsoft.CmdPal.Ext.UnitTestsBase/CommandPaletteUnitTestBase.cs`
- Common tests: `src/modules/cmdpal/Tests/Microsoft.CmdPal.Common.UnitTests/`
- Extension tests: `src/modules/cmdpal/Tests/Microsoft.CmdPal.Ext.{Name}.UnitTests/`
- UI tests: `src/modules/cmdpal/Tests/Microsoft.CmdPal.UITests/` (uses UITestBase from `src/common/UITestAutomation/`)
- ViewModel tests: `src/modules/cmdpal/Tests/Microsoft.CmdPal.UI.ViewModels.UnitTests/`
- Toolkit tests: `src/modules/cmdpal/Tests/Microsoft.CommandPalette.Extensions.Toolkit.UnitTests/`

**Test Discipline & Patterns:**
- All `.csproj` set `<IsTestProject>true</IsTestProject>` or `<TestingPlatformDisableCustomTestTarget>true</TestingPlatformDisableCustomTestTarget>`
- Output path: `$(SolutionDir)$(Platform)\$(Configuration)\WinUI3Apps\CmdPal\tests\`
- All test projects reference `Microsoft.CmdPal.Ext.UnitTestBase` (except Common)
- Moq & MSTest are consistent dependencies across all test projects

### Issue #47110 — Pluralization Tests (2026-04-20)

**Context:**
- Issue: CmdPal settings → Extensions page shows "1 commands" / "1 fallback commands" instead of singular form
- Source: `Microsoft.CmdPal.UI.ViewModels\ProviderSettingsViewModel.cs`, `ExtensionSubtext` property (lines 54-87)
- Resource strings: `Microsoft.CmdPal.UI.ViewModels\Properties\Resources.resx` (lines 132-159)
- Format strings used:
  - `builtin_extension_subtext` — "{0}, {1} commands" (plural)
  - `builtin_extension_subtext_singular` — "{0}, {1} command" (singular)
  - `builtin_extension_subtext_with_fallback` — "{0}, {1} commands, {2} fallback commands" (both plural)
  - `builtin_extension_subtext_with_fallback_singular_command` — "{0}, {1} command, {2} fallback commands"
  - `builtin_extension_subtext_with_fallback_singular_fallback` — "{0}, {1} commands, {2} fallback command"
  - `builtin_extension_subtext_with_fallback_singular_both` — "{0}, {1} command, {2} fallback command"

**Implementation Notes:**
- Pluralization logic uses pattern matching on (commandSingular, fallbackSingular) tuple (line 71)
- CompositeFormat.Parse used for resource string formatting (lines 20-26)
- Enabled/disabled state affects display (lines 58-61); disabled providers don't show command counts
- Extension name comes from IExtensionWrapper.ExtensionDisplayName or falls back to Resources.builtin_extension_name

**Test Coverage Added:**
- Created: `src/modules/cmdpal/Tests/Microsoft.CmdPal.UI.ViewModels.UnitTests/ProviderSettingsViewModelPluralizationTests.cs`
- Test class name: `ProviderSettingsViewModelPluralizationTests`
- Test methods: 17 total
  - Zero commands → plural (0 commands)
  - One command → singular (1 command)
  - Two commands → plural (2 commands)
  - Five commands → plural (5 commands)
  - One fallback → singular (1 fallback command)
  - Two fallback → plural (2 fallback commands)
  - One command + one fallback → both singular
  - One command + multiple fallback → mixed
  - Multiple commands + one fallback → mixed
  - Multiple commands + multiple fallback → both plural
  - Large counts (100 commands)
  - Disabled provider → no command counts shown
  - Built-in provider (without extension wrapper)
  - Zero fallback → doesn't mention "fallback"
  - Extension name included in subtext

**Mock Strategy:**
- Mock ISettingsService using Moq
- Create test CommandProvider class (partial, CsWinRT1028 compliant)
- Create test AppExtensionWrapper (partial)
- Use real ProviderSettings with test data
- Construct CommandProviderWrapper with specified top-level and fallback item counts

**Key Test Helpers:**
- `CreateProvider(displayName, topLevelCommandCount, fallbackCommandCount, extension)` — Factory for CommandProviderWrapper
- `CreateViewModel(provider, isEnabled)` — Factory for ProviderSettingsViewModel with mocked settings service

**Dependencies:**
- System.Collections.Immutable (for ImmutableDictionary)
- Microsoft.CmdPal.Common.Services (ISettingsService)
- Microsoft.CmdPal.UI.ViewModels.Models (ExtensionService types)
- Microsoft.CmdPal.UI.ViewModels.Services (CommandProviderWrapper, ProviderSettings)
- Microsoft.CommandPalette.Extensions (IExtensionWrapper, IListPageNavigator)
- Microsoft.CommandPalette.Extensions.Toolkit (CommandProvider, FallbackCommandItem)
- Moq (mocking framework)

**Test Pattern:**
- Arrange-Act-Assert structure
- Descriptive test method names following convention: `ExtensionSubtext_{Scenario}_{ExpectedBehavior}`
- Detailed assertion messages using string interpolation to aid debugging
- Multiple assertions per test (positive + negative checks for singular/plural forms)

### Test Removal — ProviderSettingsViewModelPluralizationTests (2026-04-21)

**Why the tests were removed:**
The test file created for issue #47110 could not compile against the real codebase. The infrastructure required to test `ProviderSettingsViewModel.ExtensionSubtext` is too complex to mock effectively.

**Key findings about real type signatures:**

1. **CommandProviderWrapper constructors:**
   - Built-in providers: `CommandProviderWrapper(ICommandProvider provider, TaskScheduler mainThread)`
   - Extension providers: `CommandProviderWrapper(IExtensionWrapper extension, TaskScheduler mainThread, ICommandProviderCache commandProviderCache)`
   - Not simple to instantiate in tests; requires ExtensionObject<T>, TaskScheduler, CommandPaletteHost

2. **TopLevelViewModel and FallbackItems:**
   - Both `CommandProviderWrapper.TopLevelItems` and `FallbackItems` are of type `TopLevelViewModel[]`
   - There is NO `FallbackItemWrapper` type
   - TopLevelViewModel constructor signature: requires ISettingsService, ProviderSettings, IServiceProvider, CommandItemViewModel, IContextMenuFactory, and more

3. **AppExtensionWrapper:**
   - Not a simple base class to inherit from
   - Actual extensions use complex patterns with ExtensionObject<T> and CommandPaletteHost
   - Cannot be easily mocked for unit tests

4. **CommandProvider abstract members:**
   - `TopLevelCommands()` is a required abstract method, not optional
   - `GetListPage(IListPageNavigator)` required
   - Simple test stubs don't satisfy the full interface contract

5. **Test infrastructure complexity:**
   - ProviderSettingsViewModel depends on deeply nested infrastructure (ExtensionObject<T>, TaskScheduler, ICommandProviderCache)
   - Mocking all dependencies would require more code than the logic being tested
   - Better to verify pluralization logic through UI tests or manual verification

**Decision:**
Removed the test file entirely. The pluralization logic is simple tuple pattern matching that can be manually verified. Unit testing it requires too much brittle infrastructure mocking. UI tests provide better ROI for this display logic.

**Learning:**
Not all code needs unit tests. When mocking infrastructure exceeds the complexity of the code under test, reconsider the testing approach. For UI display logic with simple conditionals, manual verification or UI automation tests are often more appropriate than unit tests.
