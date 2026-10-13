# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2026-10-13

The placeholder becomes a real plugin with its first skill. The Filament developer agent and a
code-patterns skill come later.

### Added

- **Skill `wame-filament-ui`** — UI/UX decisions for Laravel Filament admin panels (v3, v4, v5):
  what a screen shows, in what order, how it reads and looks. Adds the Filament specifics to the
  general method in `design-agents:wame-ui-ux` and loads that skill when it is installed.
- **Version detection first** — the Filament major is read from `composer.lock`, with a table of
  what differs for UI work: unified schemas and namespace moves in v4+, one-column `Section` /
  `Grid` / `Fieldset`, deferred table filters, `recordActions()` / `toolbarActions()`, Tailwind v4
  CSS-only themes, and the APIs that exist only in later 4.x/5.x minors.
- **Screen rules** — panel look (one brand colour, logo for both modes, font provider, `spa()`,
  dark mode, sidebar), navigation (groups, sort, actionable badges, clusters, record
  sub-navigation, global search), resource pages (simple, view and infolist, list tabs, relation
  managers), forms and schemas (sections, tabs, wizards, `helperText()` vs `hint()`, `live()` only
  when needed), tables (≤ ~7 columns, badges, descriptions, toggleable columns, filter layouts, row
  vs bulk actions, empty states), actions and modals (confirmations that state the consequence,
  slide-overs), notifications, widgets (one question per widget, polling off by default), custom
  themes and custom pages built from Filament components.
- **Accessibility and UI text** — Filament-specific accessibility (labels, icon buttons, helper
  text wiring, colour never alone), every string through translations, Slovak and Czech title-case
  fix, and locale formats in place of Filament's USD and `M j, Y` defaults.
- **Generic / AI look in Filament** — what to avoid (gradient hero widgets, default KPI cards with
  trend arrows, unjustified theme overrides, emoji, a card around every section, too many status
  colours), each with its Filament replacement.
- **Reference `reference/filament-screens.md`** — the detail behind every rule, with short
  snippets per major.

## [0.0.1] - 2026-07-24

Placeholder that reserved the plugin name; shipped nothing.
