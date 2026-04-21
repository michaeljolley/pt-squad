# Squad Team

> Microsoft Command Palette (CmdPal) — PowerToys

## Coordinator

| Name | Role | Notes |
|------|------|-------|
| Squad | Coordinator | Routes work, enforces handoffs and reviewer gates. |

## Members

| Name | Role | Charter | Status |
|------|------|---------|--------|
| Hawk | Lead | .squad/agents/hawk/charter.md | 🏗️ Active |
| Scarlett | UI Dev | .squad/agents/scarlett/charter.md | ⚛️ Active |
| Snake Eyes | Extensions Dev | .squad/agents/snake-eyes/charter.md | 🔧 Active |
| Flint | Tester | .squad/agents/flint/charter.md | 🧪 Active |
| Scribe | Scribe | .squad/agents/scribe/charter.md | 📋 Active |
| Ralph | Work Monitor | — | 🔄 Monitor |

## Project Context

- **Owner:** Michael Jolley
- **Project:** Microsoft Command Palette (CmdPal) — a PowerToys utility providing a command launcher with extensible plugin architecture
- **Stack:** C# (WinUI 3 / WinAppSDK), C++/WinRT, XAML, WinRT IDL
- **Scope:** `src/modules/cmdpal/` and solution filter `CommandPalette.slnf` only
- **Created:** 2026-04-20

### Architecture Overview

| Layer | Location | Description |
|-------|----------|-------------|
| Core UI | `Microsoft.CmdPal.UI/` | WinUI 3 app, XAML pages, controls |
| ViewModels | `Microsoft.CmdPal.UI.ViewModels/` | MVVM view models |
| Common | `Microsoft.CmdPal.Common/` | Shared utilities, helpers |
| Extension SDK | `extensionsdk/Microsoft.CommandPalette.Extensions/` | WinRT IDL interface definitions |
| Extension Toolkit | `extensionsdk/Microsoft.CommandPalette.Extensions.Toolkit/` | C# helper classes for extension authors |
| Built-in Extensions | `ext/` | 17 built-in extensions (Apps, Calc, Bookmarks, etc.) |
| Native | `CmdPalKeyboardService/`, `CmdPalModuleInterface/`, `Microsoft.Terminal.UI/` | C++ keyboard hooks, module interface, terminal rendering |
| Tests | `Tests/` | 16 test projects (unit, UI, extension tests) |
