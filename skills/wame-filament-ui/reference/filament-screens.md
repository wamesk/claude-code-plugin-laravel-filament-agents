# Filament screens — detail

Decision rules with the API names per major. Snippets use v4/v5 syntax unless marked v3; they
show the API shape only — every string a user sees goes through `__()` (section 13).

## Contents

1. Version map
2. Panel look
3. Navigation
4. Resource pages
5. Forms and schemas
6. Tables
7. Actions and modals
8. Notifications
9. Widgets and dashboards
10. Custom theme
11. Custom pages
12. Accessibility specifics
13. UI text and locale
14. Generic / AI look in Filament

## 1. Version map

```bash
grep -A1 '"name": "filament/filament"' composer.lock   # next line: "version": "v4.x.y"
```

| | v3 | v4 and v5 |
|---|---|---|
| Resource form / infolist | `form(Form $form)`, `infolist(Infolist $infolist)`, `->schema([...])` | `form(Schema $schema)`, `infolist(Schema $schema)`, `->components([...])` |
| Layout classes | `Filament\Forms\Components\…` and `Filament\Infolists\Components\…` | `Filament\Schemas\Components\…` for both; `Split` → `Flex` |
| Tabs, steps | `Forms\Components\Tabs\Tab`, `Wizard\Step`; list tabs `Filament\Resources\Components\Tab` | `Filament\Schemas\Components\Tabs\Tab` (list tabs too), `Wizard\Step` |
| `Get` in closures | `Filament\Forms\Get` | `Filament\Schemas\Components\Utilities\Get` |
| Actions | `Filament\Tables\Actions\*` in tables, `Filament\Actions\*` on pages, `Filament\Notifications\Actions\Action` | `Filament\Actions\*` everywhere |
| Table action slots | `actions()`, `bulkActions()`, `actionsPosition()` | `recordActions()`, `toolbarActions()`, `recordActionsPosition()` |
| Action modal form | `->form([...])` | `->schema([...])` |
| Widths | `Filament\Support\Enums\MaxWidth` | `Filament\Support\Enums\Width` |
| Icons | strings (`'heroicon-o-users'`) | `Filament\Support\Icons\Heroicon` enum (strings still work) |
| Navigation group | `?string $navigationGroup` | `string \| UnitEnum \| null`; enum groups with `HasLabel`/`HasIcon` |
| Sub-navigation enum | `Filament\Pages\SubNavigationPosition` | `Filament\Pages\Enums\SubNavigationPosition` |
| Page view, chart heading, polling | `protected static` properties | instance properties (`protected string $view`, `protected ?string $heading`, `protected ?string $pollingInterval`) |
| Format defaults | statics `Table::$defaultCurrency`, `Infolist::$defaultDateDisplayFormat` … | `Table::configureUsing()` / `Schema::configureUsing()` with `defaultCurrency()` … |

v5 uses the v4 names above; only Livewire code in custom pages and widgets changes (Livewire 4).

## 2. Panel look

```php
return $panel
    ->colors(['primary' => '#0f766e'])           // one brand hex; Filament builds the 11 shades
    ->brandLogo(asset('images/logo.svg'))
    ->darkModeBrandLogo(asset('images/logo-dark.svg'))
    ->brandLogoHeight('2rem')
    ->favicon(asset('images/favicon.svg'))
    ->spa()
    ->sidebarCollapsibleOnDesktop()
    ->maxContentWidth(Width::Full)               // v3: MaxWidth::Full
    ->unsavedChangesAlerts();
```

- Shades are chosen for contrast "where possible", not guaranteed: with a light brand colour
  (yellow, lime, sky) check the button label contrast in both modes, or use a darker brand shade.
- `brandName()` only when there is no logo. Always give the dark-mode logo when the logo is dark.
- Default font Inter via Bunny Fonts (GDPR-compliant CDN). `font('…', provider: GoogleFontProvider::class)`
  or `LocalFontProvider` with a `url:` for self-hosted brand fonts.
- Theme mode follows the system. `defaultThemeMode(ThemeMode::Light)` when the brand is designed
  light; `darkMode(false)` is the last resort. Check every custom colour in both modes.
- `spa()` uses `wire:navigate`: list URLs of pages whose third-party JS (maps, charts, editors)
  does not re-initialise in `spaUrlExceptions()` (wildcards allowed, closure for `getUrl()`).
- Collapsed sidebar works best with group icons (the icon opens a dropdown of the group).
- `maxContentWidth()` full for table-heavy panels; keep the default for form-heavy ones — long
  lines are harder to read.

## 3. Navigation

- Order: ungrouped items (dashboard) first, then groups in `navigationGroups()` order, then
  `$navigationSort` inside a group. Name groups by domain or job, not "Resources". "Settings" last.
- v4+ enum groups keep label, icon and order in one place:

```php
enum NavigationGroup implements HasLabel
{
    case Billing;
    case Settings;

    public function getLabel(): string
    {
        return __('navigation.group.' . Str::snake($this->name));
    }
}
// on the resource: protected static string | UnitEnum | null $navigationGroup = NavigationGroup::Billing;
```

  v3: `NavigationGroup::make()->label(fn (): string => __('…'))->icon(…)->collapsed()` in
  `navigationGroups()`, and `getNavigationGroup()` returning the same translated label.
- Items without icons in one group are joined by a vertical line — a clean tree for long groups.
- Child item: `$navigationParentItem` (same group). A third level → cluster instead.
- Badges: `getNavigationBadge()` returns the count as a string or `null` (no badge at zero);
  `getNavigationBadgeColor()` `danger` for overdue, `warning` for waiting; `getNavigationBadgeTooltip()`.
- Clusters (`make:filament-cluster`, `$cluster` on the resource or page): members leave the main
  navigation and share a sub-navigation; the cluster name joins the breadcrumbs.
- Record sub-navigation for a record with several jobs:

```php
public static function getRecordSubNavigation(Page $page): array
{
    return $page->generateNavigationItems([ViewContract::class, EditContract::class, ManageContractItems::class]);
}
```

- Global search: `$recordTitleAttribute`, `getGloballySearchableAttributes()`,
  `getGlobalSearchResultDetails()` (eager-load in `getGlobalSearchEloquentQuery()`),
  `globalSearchKeyBindings(['command+k', 'ctrl+k'])`. `$isGloballySearchable = false` on resources
  nobody looks up.
- Reachability: a resource or page with `$shouldRegisterNavigation = false` must still be reachable
  from its parent (relation manager, sub-navigation, action URL).

## 4. Resource pages

| Record | Pages |
|---|---|
| ≤ ~5 fields, no relations, edited rarely | `--simple`: one list page, create/edit in modals |
| Normal CRUD | list, create, edit |
| Has status, lifecycle, computed values or is mostly read | add `--view` with an infolist; editing is an action |

- Infolist: the facts that drive the next decision on top (status `badge()`, amounts, dates), then
  sections. `inlineLabel()` for dense key–value blocks; `placeholder('—')` for empty values;
  `copyable()` for codes the user pastes elsewhere (variable symbol, IBAN).
- Never show system fields (ids, created-by) in the form; put them at the end of the infolist.
- Headings: set `$recordTitleAttribute` or `getRecordTitle()` so titles and breadcrumbs name the
  record ("Faktúra 2026-0012"), not just the model.
- Header actions (`getHeaderActions()`): one filled primary; the rest `color('gray')` or in an
  `ActionGroup`; delete last.
- List tabs:

```php
public function getTabs(): array
{
    return [
        'all' => Tab::make(__('invoice.tab.all')),
        'unpaid' => Tab::make(__('invoice.tab.unpaid'))
            ->modifyQueryUsing(fn (Builder $query) => $query->whereNull('paid_at'))
            ->badge(fn (): int => Invoice::whereNull('paid_at')->count())
            ->badgeColor('warning'),
    ];
}
```

- Relations: relation managers under the form; `hasCombinedRelationManagerTabsWithContent()` puts
  the form and 3+ relations into one tab bar; a heavy relation gets a `ManageRelatedRecords` page in
  the record sub-navigation. Resource stats belong in `getHeaderWidgets()` of the list page, not
  on the dashboard.

## 5. Forms and schemas

```php
return $schema->components([
    Section::make(__('contract.section.parties'))
        ->columns(2)
        ->columnSpanFull()                                  // v4+: sections take one column otherwise
        ->schema([
            Select::make('customer_id')->label(__('contract.field.customer'))
                ->relationship('customer', 'name')->searchable()->preload(),
            DatePicker::make('valid_from')->label(__('contract.field.valid_from'))
                ->helperText(__('contract.help.valid_from')),
            Textarea::make('note')->label(__('contract.field.note'))->columnSpanFull(),
        ]),
    Section::make(__('contract.section.advanced'))
        ->collapsible()->collapsed()->persistCollapsed()
        ->columnSpanFull()
        ->schema([/* rarely used fields */]),
]);
```

v3: same calls with `Filament\Forms\Components\Section`, `form(Form $form)` and `$form->schema([...])`.

- Order: what it is (identity) → what the user decides (values) → optional and advanced (collapsed
  section). Mirror the paper document the user copies from, when there is one.
- 1–3 related fields → `Grid`, `Group` or `Fieldset`, not another `Section` card.
- `aside()` puts heading and description beside the fields — good for settings pages, wasteful in
  dense data entry.
- `Tabs` hide content: do not put required create fields behind a non-first tab;
  `persistTabInQueryString()` keeps the tab on reload and in shared links.
- `Wizard` for create flows of 3–5 steps where later steps depend on earlier ones (`HasWizard` +
  `getSteps()` on a create page); `skippable()` when steps are independent;
  `persistStepInQueryString()`.
- Text around a field: `helperText()` = the instruction needed to fill it (wired to the input for
  screen readers on v4/v5); `hint()` / `hintIcon($icon, tooltip: …)` = short side note, not
  announced with the input; placeholder = format example. v4+ `belowContent()` / `aboveContent()`
  for richer content.
- Choice controls: `Toggle` = on/off with immediate meaning; `Checkbox` = consent;
  `ToggleButtons` = 2–5 options shown at once (takes enum label, colour and icon); `Radio` with
  `descriptions()` when options need explaining; `Select` `searchable()` + `preload()` for small
  relations, `searchable()` alone for large ones.
- Units and currency in `prefix()`/`suffix()` in the locale's order (Slovak: suffix `€`).
- Reactivity costs a request per change: `live()` only when something must change;
  `live(onBlur: true)` for typing, `live(debounce: 500)` for search-as-you-type. v4+:
  `visibleJs()`/`hiddenJs()` for pure show/hide, `partiallyRenderComponentsAfterStateUpdated([...])`
  or `skipRenderAfterStateUpdated()` to avoid re-rendering the whole form.

## 6. Tables

```php
return $table
    ->columns([
        TextColumn::make('number')->label(__('invoice.column.number'))
            ->searchable()->sortable()->copyable(),
        TextColumn::make('customer.name')->label(__('invoice.column.customer'))
            ->description(fn (Invoice $record): ?string => $record->premise?->name)->searchable(),
        TextColumn::make('status')->label(__('invoice.column.status'))->badge(),   // enum HasLabel + HasColor
        TextColumn::make('total')->label(__('invoice.column.total'))->money()->alignEnd()->sortable(),
        TextColumn::make('due_at')->label(__('invoice.column.due_at'))->date()->sortable(),
        TextColumn::make('created_at')->label(__('invoice.column.created_at'))->dateTime()
            ->toggleable(isToggledHiddenByDefault: true),
    ])
    ->defaultSort('due_at', 'desc')
    ->filters([
        SelectFilter::make('status')->label(__('invoice.filter.status'))->options(InvoiceStatus::class)->multiple(),
    ], layout: FiltersLayout::AboveContent)
    ->filtersFormColumns(3)
    ->recordActions([
        Action::make('send')->label(__('invoice.action.send'))->icon(Heroicon::OutlinedPaperAirplane),
        ActionGroup::make([EditAction::make(), DeleteAction::make()]),
    ])
    ->toolbarActions([BulkActionGroup::make([DeleteBulkAction::make()])])
    ->emptyStateHeading(__('invoice.empty.heading'))
    ->emptyStateDescription(__('invoice.empty.description'))
    ->emptyStateActions([CreateAction::make()]);
```

v3: `->actions([...])`, `->bulkActions([...])`, actions from `Filament\Tables\Actions`, icon strings.

- Column order: what identifies the row → the 2–3 values the decision needs → status → date.
  IDs and timestamps toggleable and hidden by default.
- `description()` puts a second line under a value (e-mail under name) instead of a new column.
- Numbers and money `alignEnd()` so digits line up; totals with `summarize(Sum::make())`.
- Long text: `limit()` + `tooltip()`; `wrap()` for medium text. `since()` with `dateTimeTooltip()`
  when recency matters more than the date.
- `placeholder('—')` for empty cells — an empty cell reads as "still loading".
- Status colours mapped once on the enum (`HasColor`): gray neutral, warning waiting, danger
  overdue or failed, success done. Primary and info rarely.
- Search box: `searchable()` on columns people type (number, name, e-mail) that the database can
  search fast; never on computed or long text columns.
- Filters: `SelectFilter` for enums and relations (`relationship()`, `multiple()`),
  `TernaryFilter` for yes/no/all, `indicateUsing()` so custom filters show chips. Default values
  that match the daily job ("open only"). v4+ deferred filters suit many filters;
  `deferFilters(false)` for one or two selects.
- Tabs vs filters: tabs for 2–5 fixed daily slices, filters for ad-hoc combinations.
- The row click opens the record (`recordUrl()`); do not add View and Edit buttons to every row.
  Visible row actions only for frequent non-navigation jobs (send, approve).
- Bulk actions only for jobs done on many rows; `deselectRecordsAfterCompletion()` after them.
- `poll('10s')` only for lists that change while watched (imports, queues).
- Page size: `defaultPaginationPageOption(25)`, `paginationPageOptions([10, 25, 50])`; v4+ has
  no `all` option.
- v4+ adds the primary key as a last sort so pages stay stable; `defaultKeySort(false)` drops it.
- Narrow screens: `visibleFrom('md')` on secondary columns, `stackedOnMobile()` (later minors),
  or `Split`/`Stack` layouts for card-like rows. `striped()` only for wide numeric tables.

## 7. Actions and modals

```php
DeleteBulkAction::make()
    ->modalHeading(__('reading.delete.heading'))
    ->modalDescription(fn (Collection $records): string => __(
        'reading.delete.description',
        ['count' => $records->count()],
    ))
    ->modalSubmitActionLabel(__('reading.delete.submit'));
// sk description: "Zmaže sa 12 odpočtov za máj a spotreba meradiel sa prepočíta. Nedá sa to vrátiť."
```

- Labels are verb + object ("Odoslať faktúru"), never "OK" or "Submit". Heading = the action,
  description = consequence + count + whether it can be undone.
- Built-in `DeleteAction`/`ForceDeleteAction` confirm already — override the description to name
  the record and the consequence. Reversible actions need no confirmation.
- `requiresConfirmation()` does nothing on an action with `url()` — redirect inside `action()`.
- Colours: one primary per area, `gray` for secondary, `danger` only for destructive.
- `modalWidth()` with `Width` (v3 `MaxWidth`); `slideOver()` for long forms or side detail;
  `stickyModalFooter()` so the submit stays visible in long modals.
- Authorization: `visible()`/`hidden()`/`authorize('update')`. v4+ `authorizationTooltip()`
  disables the action and shows the policy's denial message — use it where users would otherwise
  search for a missing button.
- `keyBindings(['command+s', 'ctrl+s'])` only for the one or two actions used all day.
- `iconButton()` in dense rows; keep the label (accessible name) and add `tooltip()`.

## 8. Notifications

```php
Notification::make()
    ->title(__('invoice.notification.sent'))
    ->success()
    ->actions([
        Action::make('view')->label(__('invoice.action.view'))->url($url), // v3: Filament\Notifications\Actions\Action
    ])
    ->send();
```

- Success: title only, past tense, the object named. Body only when it adds information.
- Failure: what failed and what the user can do; `danger()` + `persistent()` so it is not missed.
- Built-in actions notify already: `successNotificationTitle()` / `successNotification()` to reword,
  never a second notification.
- Queued or long jobs: `sendToDatabase($user)` with an action to the result; panel
  `databaseNotifications()` (+ `databaseNotificationsPolling()`).
- Validation errors stay next to the field; never a toast.
- Undo for reversible bulk changes = a notification action, instead of a confirmation before.

## 9. Widgets and dashboards

- Top of the dashboard: what needs action now (a `TableWidget` of overdue, expiring, waiting rows).
  Then counts that link to the filtered list. Charts last, only for trends that change decisions.
- `Stat::make($label, $value)->description()->url()` — each stat opens the list behind it. Trend
  arrows (`descriptionIcon()`) and `color()` only with a real comparison period and meaning.
  `chart()` sparklines only when the trend is the point.
- `$pollingInterval` (5 s default on stats) → `null` unless live; widgets are lazy by default
  (`$isLazy`); `canView()` mirrors the policy of the data shown.
- Layout: `$sort`, `$columnSpan`, dashboard `getColumns()`; dashboard filters with
  `filtersForm()` + `InteractsWithPageFilters` only when every widget uses them.
- Remove `FilamentInfoWidget` (and usually `AccountWidget`) from the panel's `widgets()`.

## 10. Custom theme

- Only for: a brand need `colors()`/`font()`/logo cannot meet, Tailwind classes in your own Blade
  (required on v4+), or targeted density tweaks.
- v3: `make:filament-theme` → `theme.css` + `tailwind.config.js` (Tailwind v3), registered with
  `viteTheme()`. With Tailwind v4 in the project, v3 themes need the Tailwind v3 CLI.
- v4/v5: `make:filament-theme` → a CSS file importing
  `vendor/filament/filament/resources/css/theme.css` with `@source` lines for your Blade and PHP
  paths (Tailwind v4, no config file), registered with `viteTheme()`.
- Style `fi-*` CSS hook classes. Do not publish Filament's views — every update can break them.

## 11. Custom pages

- `make:filament-page`, `canAccess()`, translated `getTitle()` and `getSubheading()`, actions in
  `getHeaderActions()`, widgets in `getHeaderWidgets()`.
- v4/v5: compose the page in `content(Schema $schema)` from schema components (`Section`,
  `Text`, `Actions`, `EmbeddedTable` for a page that has a table) — no Blade, no theme needed.
- Blade (v3, or v4+ when needed): wrap in `<x-filament-panels::page>`; use
  `<x-filament::section>`, `<x-filament::button>`, `<x-filament::badge>`, `<x-filament::link>`,
  `<x-filament::modal>`, `<x-filament::tabs>`, `<x-filament::loading-indicator>`; render forms
  and tables with `{{ $this->form }}` / `{{ $this->table }}`. On v4+ any Tailwind class you add
  needs a custom theme.

## 12. Accessibility specifics

- `helperText()` is linked to the input (v4/v5); `hint()` is not — required instructions go in
  `helperText()`.
- `hiddenLabel()` renders the label screen-reader-only; `inlineLabel()` keeps it visible.
- Icon buttons use the action label as their accessible name — never pass an empty label.
- Confirmation modals: v5 renders them as `alertdialog`, and the description is read on open —
  write it as the consequence.
- Badges carry text; never a colour-only dot. Boolean `IconColumn` only where the column label
  says what yes/no means.
- Tooltips hold extra detail, never essential information (touch users cannot hover).
- Filament sets `<html lang>` from the app locale and ships `sk`/`cs` strings — set the locale,
  do not translate Filament's own UI by hand.
- Hand-built modals, dropdowns and tabs lose Filament's focus management — use the components.

## 13. UI text and locale

- Every label explicit: `->label(__('invoice.field.number'))`. Auto-generated labels are English
  and title-cased; `translateLabel()` uses that auto label as a JSON key — skip it.
- Resource and page texts via methods, not static properties: `getModelLabel()`,
  `getPluralModelLabel()`, `getNavigationLabel()`, `getNavigationGroup()`, `getTitle()`; widget
  `getHeading()`; enum `getLabel()` returning `__()`.
- Slovak and Czech: `protected static bool $hasTitleCaseModelLabel = false;`.
- Publish `lang/vendor/filament-*` only to change a specific Filament wording.
- Formats: defaults are USD and `M j, Y`. v4+ in a service provider:

```php
Table::configureUsing(fn (Table $table) => $table
    ->defaultCurrency('EUR')->defaultNumberLocale('sk')->defaultDateDisplayFormat('j. n. Y'));
Schema::configureUsing(fn (Schema $schema) => $schema
    ->defaultCurrency('EUR')->defaultNumberLocale('sk')->defaultDateDisplayFormat('j. n. Y'));
```

  v3: `Table::$defaultCurrency = 'eur';` and the same statics on `Infolist`
  (`$defaultNumberLocale`, `$defaultDateDisplayFormat`, `$defaultDateTimeDisplayFormat`).

## 14. Generic / AI look in Filament

What to use instead of the six tells listed in `SKILL.md`:

- hero or welcome widget → a table widget of what needs action;
- KPI row → counts that link to lists; trend arrows only with a real comparison;
- theme override → brand colour + logo;
- emoji → Heroicons only where they aid recognition;
- a card per section → fewer sections; `Grid`/`Group`/`Fieldset` for small groups;
- many status colours → ≤ 4–5 semantic colours mapped once on the enum.

Four more tells:

| Tell | Instead |
|---|---|
| `prefixIcon()` or `hintIcon()` on every field | icons only where they identify the input (e-mail, phone, search) |
| View/Edit/Delete buttons on every row | row click + one `ActionGroup` |
| Chart for a single number | the number, linked to its list |
| Placeholder repeating the label | a format example or nothing |

Full list: `design-agents:wame-ui-ux` → `reference/ai-design-tells.md`.
