# Bigtop

Lightweight application shell framework for building panel-based desktop/web apps with IDE-like chrome, without being an IDE.

## Packages

| Package | Description |
| --- | --- |
| `@bigtop/core` | Shell, commands, settings, theming, keybindings, notifications |
| `@bigtop/dockview` | Dockview integration, panel registry, layout persistence |
| `@bigtop/shadcn` | UI primitives (Radix-based, copy-paste from shadcn/ui) |

## Tech Stack

React 19, Tailwind v4, dockview, cmdk, tinykeys, sonner, Lucide

## Development

```bash
moon run :build          # Build all packages
moon run :typecheck      # Type check all packages
moon run :lint           # Lint all packages
moon run :test           # Test all packages
```

See `docs/` for design documents.