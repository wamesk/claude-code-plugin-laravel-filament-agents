# laravel-filament-agents

UI/UX rules for Laravel Filament admin panels for Claude Code, by [WAME](https://wame.sk). Part of the `wame` marketplace.

## Installation

```
/plugin marketplace add wamesk/claude-code
/plugin install laravel-filament-agents@wame
```

## Skills

| Skill | What it holds |
|-------|---------------|
| **wame-filament-ui** | UI/UX decisions for Filament panels (v3, v4, v5; WAME projects use 5.x): what a screen shows, in what order, how it reads and looks. Reads the Filament major from `composer.lock` first and names what differs per major (unified schemas and namespace moves in v4+, Tailwind v4 themes in v4+). Covers the panel look, navigation, resource pages, forms and schemas, tables, actions and modals, notifications, widgets, custom themes and custom pages, accessibility, translated UI text, and the generic "AI look" to avoid. The detail lives in `reference/filament-screens.md`. |

## Status

`0.1.0` ships the UI/UX skill only. A Filament developer agent and a code-patterns skill come
later, written from WAME's first real Filament work rather than from documentation alone.

For Laravel (non-Filament) and Laravel Nova work, use
[`laravel-agents`](https://github.com/wamesk/claude-code-plugin-laravel-agents) and
[`laravel-nova-agents`](https://github.com/wamesk/claude-code-plugin-laravel-nova-agents).

## How it composes with design-agents

[`design-agents`](https://github.com/wamesk/claude-code-plugin-design-agents) holds the general
method in its `wame-ui-ux` skill — user goal, heuristics, WCAG 2.2 AA, states, forms, tables and
the AI-look list. When a project has `filament/filament` in `composer.lock`, `wame-ui-ux` loads
`laravel-filament-agents:wame-filament-ui` for the platform rules; where the two disagree, the
Filament skill wins.

`wame-filament-ui` also works alone: with `design-agents` installed it loads `wame-ui-ux` for the
general method, without it it applies the Filament rules only.

## License

MIT
