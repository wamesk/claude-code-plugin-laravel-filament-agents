---
name: wame-filament-ui
description: "Guides UI/UX decisions in Laravel Filament admin panels (v3, v4, v5): what a screen shows, in what order and how it reads — panel theme, navigation, resource pages, forms, tables, actions, notifications, widgets, custom pages — with the per-major differences. Use when designing, building or reviewing a Filament screen. Triggers: 'navrhni Filament obrazovku', 'rozlož formulár vo Filamente', 'uprav tabuľku vo Filamente', 'review Filament UX'."
---

# WAME Filament UI

Filament specifics for what a panel screen shows, in what order, how it reads and looks. The
general method (user goal, heuristics, WCAG 2.2 AA, states, forms, tables, the AI-look list) is
`design-agents:wame-ui-ux`: load it when installed; this skill wins where they disagree. Class
structure and code patterns are out of scope: code follows the project's sibling resources and
pages (this plugin has no code-patterns skill yet). Detail: `reference/filament-screens.md`.

## 1. Read the major first

Find `"name": "filament/filament"` in `composer.lock` and read its `version`. Name APIs for that
major only, follow the sibling resources, never mix v3 and v4+ namespaces.

| UI concern | v3 | v4 and v5 |
|---|---|---|
| Layout components | `Filament\Forms\Components\Section` (+ `Infolists\…` copies) | one `Filament\Schemas\Components\Section`; `Split` → `Flex` |
| `Section`, `Grid`, `Fieldset` | span the full parent grid | span one column — add `columnSpanFull()` |
| `columnSpan(2)` | every breakpoint | `lg` and up |
| Table actions | `actions()`, `bulkActions()` | `recordActions()`, `toolbarActions()`, all in `Filament\Actions` |
| Form in an action modal | `form([...])` | `schema([...])` |
| Table filters | apply on change | deferred behind Apply (`deferFilters(false)` undoes) |
| Custom theme | Tailwind v3 + `tailwind.config.js`; Tailwind classes in own Blade work without it | Tailwind v4, CSS with `@source`; own Blade classes do nothing without it |
| Unavailable action | hide it | `authorizationTooltip()` disables and explains |

v5 is the v4 API on Livewire 4 (its upgrade guide adds requirements only). Some APIs arrived in
later 4.x/5.x minors (`Callout`, `EmptyState`, `stackedOnMobile()`, `globalSearchResourceOptIn()`,
`GlobalSearchPosition::Sidebar`): confirm them in `vendor/filament/` before naming them.

## 2. Panel look

- Keep Filament's design system. Brand = `colors()` with one `primary`, `brandLogo()` +
  `darkModeBrandLogo()` + `brandLogoHeight()`, `favicon()`, maybe `font()`. Usually that is all.
- Keep `danger`, `warning`, `success`, `info`, `gray` semantic; never use the brand colour as a
  status. Check text contrast on the primary button in light and dark mode.
- `font()` loads from Bunny Fonts; use `LocalFontProvider` when third-party requests are not allowed.
- Keep dark mode; switch it off (`darkMode(false)`) only when the brand assets cannot support it.
- `spa()` for faster navigation; pages with third-party JS go to `spaUrlExceptions()`.
- `sidebarCollapsibleOnDesktop()` for data-heavy panels; `topNavigation()` only for ≤ 6 ungrouped items.
- `maxContentWidth()` full width for wide tables; `unsavedChangesAlerts()` when forms are long.

## 3. Navigation

- Groups follow the user's jobs, ordered by frequency, declared in one place (`navigationGroups()`,
  or on v4+ an enum with `HasLabel`/`HasIcon`). More than ~7 items in a group → a cluster.
- Give icons to the group or to its items, not both — the expanded sidebar hides item icons of an icon group.
- Badge only counts that need action ("12 to approve"), coloured by urgency, with a tooltip saying
  what is counted. It runs on every page load: keep the query cheap or cache it.
- Cluster = self-contained area (settings, catalogue). `getRecordSubNavigation()` = one record with
  several pages; `SubNavigationPosition::Top` renders them as tabs.
- Global search only for records looked up by name or number: `$recordTitleAttribute`, the searched
  columns, 1–2 result details. A resource without an Edit or View page returns no results.

## 4. Resource pages

- ≤ ~5 fields and no relations → simple modal resource (`make:filament-resource --simple`).
- Add a View page with an infolist when users mostly read the record or it carries computed or
  status data. A disabled form is not a read view.
- One primary header action per page; the rest grey or in an `ActionGroup`.
- List tabs (`getTabs()`) for the 2–5 slices people switch between daily; a badge only on the
  "needs action" tab — each badge is a count query.
- Related data: relation manager = table under the record; relation page = a job of its own;
  `Repeater` = a few child rows edited together with the parent.

## 5. Forms

- Field order follows the user's mental model or the source document, never the migration.
- `Section` with a heading per topic; `description()` only when it helps; `aside()` for
  settings-style forms. `Tabs` only for independent parts. `Wizard` only for create flows whose
  later steps depend on earlier answers.
- Short related fields share a row (`columns(2)`); long text `columnSpanFull()`.
- `helperText()` = instruction needed to fill the field (screen readers get it on v4/v5).
  `hint()` = short side note. Placeholder = format example only, never the label.
- Units and currency in `prefix()`/`suffix()`. 2–5 choices → `ToggleButtons` or `Radio` with
  `descriptions()`; more → `Select`.
- `live()` only when another field or the layout must change; `live(onBlur: true)` for text.
  v4+: pure show/hide via `visibleJs()`/`hiddenJs()` (no request).

## 6. Tables

- ≤ ~7 visible columns. First column = what people recognise the row by. Status as `badge()`
  with colour + label from the enum. Merge pairs with `description()`. Secondary columns
  `toggleable(isToggledHiddenByDefault: true)`.
- `searchable()` only on columns people type into the search box and the database can search
  fast; `sortable()` where order matters; always a `defaultSort()`.
- Money, dates and numbers in the user's locale — Filament defaults to USD and `M j, Y`; set them once.
- Filters: ≤ 3 → default dropdown; used on most visits → `FiltersLayout::AboveContent` +
  `filtersFormColumns()`; daily work lists → `persistFiltersInSession()`.
- Row actions: ≤ 2 visible, the rest and every destructive one in an `ActionGroup`. Bulk actions
  in a `BulkActionGroup`.
- Empty state: `emptyStateHeading()` says why it is empty, `emptyStateDescription()` the next step,
  `emptyStateActions()` the action that fills it.
- Mobile: `visibleFrom('md')` for secondary columns, or `stackedOnMobile()` where available.

## 7. Actions, modals, notifications

- Confirm only what cannot be undone: `requiresConfirmation()` + `modalDescription()` naming the
  consequence and the count, `modalSubmitActionLabel()` with the verb, `color('danger')`.
- Modal for ≤ ~6 fields; `slideOver()` for long forms or detail compared with the page behind it.
- Icon-only actions keep a translated `label()` (it is the accessible name) and get a `tooltip()`.
- Success notification = short past-tense title; failure = what failed and what to do, `persistent()`;
  long jobs → `sendToDatabase()` with an action to the result. Retitle built-in notifications
  (`successNotificationTitle()`) instead of sending a second one.

## 8. Widgets

- One question per widget, named by its heading ("Faktúry po splatnosti"); what needs action first.
- Drop the generated `FilamentInfoWidget` (and usually `AccountWidget`) from client panels.
- Stats widgets poll every 5 s by default — `$pollingInterval = null` unless the number moves live.
- `canView()` per widget. A table of what to do next beats a chart.

## 9. Theme and custom pages

- Custom theme only when the brand needs what `colors()`, `font()` and the logo cannot do; style
  through `fi-*` CSS hook classes, never by publishing Filament's views.
- Custom pages: v4+ compose `content(Schema $schema)` from schema components; Blade views use
  `<x-filament-panels::page>` and `<x-filament::…>` components (section, button, badge, modal,
  tabs, …), never raw HTML with hand-written classes.

## 10. UI text and accessibility

- No auto-generated labels: every field, column, entry, filter, action, tab, widget heading and
  notification text goes through `__()` with an English key in the module's lang file. Static
  properties cannot call `__()`: override `getModelLabel()`, `getPluralModelLabel()`,
  `getNavigationLabel()`, `getNavigationGroup()`, `getTitle()`.
- Slovak and Czech: `$hasTitleCaseModelLabel = false`, otherwise titles read "Nájomné Zmluvy".
- `hiddenLabel()` keeps the label for screen readers; never hide a label by leaving it empty.
- Colour is never the only signal: badges carry text, boolean icons only where the column label
  gives the meaning. Use Filament's modals, dropdowns and buttons — they bring focus handling.

## Generic / AI look in Filament

Avoid: gradient hero widgets; a row of KPI stat cards with green/red trend arrows as the default
dashboard; overriding the panel theme without a brand reason; emoji in labels, navigation or
notifications; wrapping every section in its own card; more than 4–5 status colours on one screen.
What to do instead, plus four more tells: reference §14. Full list: `design-agents:wame-ui-ux` →
`reference/ai-design-tells.md`.

## Pre-finish checklist

- [ ] Filament major read from `composer.lock`; every API named exists in it.
- [ ] Panel look = brand colour, logo (light + dark), maybe font; generic-look list checked.
- [ ] Every new screen reachable from the navigation or from its parent record.
- [ ] Forms follow the user's model; tables ≤ ~7 columns, default sort, designed empty state.
- [ ] One primary action per page; irreversible actions confirmed with their consequence.
- [ ] No hard-coded or auto-generated UI text; locale formats set.
