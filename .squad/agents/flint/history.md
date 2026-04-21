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
