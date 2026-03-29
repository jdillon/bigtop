# Bigtop Monorepo Scaffold Design

Spec for the initial bigtop monorepo setup: directory structure, tooling, packages, theming, and phased build plan.

## Project Type

Library project per the [Project Skeleton with Bun and Moon](obsidian://open?vault=Cirqil&file=_Users%2Fjason%2FProject%20Skeleton%20with%20Bun%20and%20Moon) template. Published packages consumed externally under `@bigtop/*` scope. Domain: bigtop.dev.

## Directory Layout

```
bigtop/
  .moon/
    workspace.yml
    toolchain.yml
  .github/
    workflows/
      ci.yml
  packages/
    core/                      # @bigtop/core
    dockview/                  # @bigtop/dockview
    shadcn/                    # @bigtop/shadcn
  tooling/
    typescript-config/         # @bigtop/typescript-config
    eslint-config/             # @bigtop/eslint-config
  sandbox/                     # Exploration apps (git-tracked)
  tmp/                         # Gitignored scratch
  docs/                        # Design docs, specs (flat)
  package.json                 # Bun workspace root — no scripts
  bunfig.toml
  tsconfig.json                # Root references
  .gitignore
```

## Moon Configuration

### .moon/workspace.yml

```yaml
projects:
  - 'packages/*'
  - 'tooling/*'
  - 'sandbox/*'
```

### .moon/toolchain.yml

```yaml
bun:
  version: '<latest-stable>'
  syncProjectWorkspaceDependencies: true
```

Pinned to latest stable bun at scaffold time (check `bun --version` or bun releases).

## Packages

### @bigtop/shadcn

Copy-paste UI primitives from shadcn/ui (Radix-based). Dedicated package so component primitives stay separate from shell logic.

Initial components: Button, Badge, Tooltip, DropdownMenu, Dialog, Input, ScrollArea.

CSS variables mapped to `--bigtop-*` so all primitives inherit the shell theme.

**Dependencies**: react (peer), tailwindcss (peer)

### @bigtop/core

Shell framework. Composable React components, not a config object.

Modules:
- **theme/**: `ThemeProvider`, `tokens.css` (`--bigtop-*` variables), dark/light presets. Handles dark/light/system modes from day one via `prefers-color-scheme` media query. Persists choice via `SettingsApi`. Adds `data-theme` attribute to root.
- **shell/**: `Shell`, `ActivityBar`, `StatusBar`, `TitleBar` — composable layout components.
- **commands/**: `CommandRegistry`, `CommandPalette` (cmdk wrapper), `useCommand` hook. Cmd+K to open.
- **keybindings/**: `KeybindingManager` (tinykeys wrapper), `useKeybinding` hook. Keybindings map to command IDs.
- **notifications/**: `Toaster` (sonner wrapper), `useNotify` hook.
- **settings/**: `SettingsApi` interface, `SettingsProvider`, `useSettings` hook, `LocalStorageSettingsApi` default implementation. Storage is pluggable per platform.
- **platform/**: `PlatformApi` interface — `openFile`, `copyToClipboard`, `openExternal`, `getSettingsApi`. Web defaults provided; Electron and VS Code adapters come later.

**Dependencies**: `@bigtop/shadcn` (workspace), cmdk, tinykeys, sonner, react (peer)

### @bigtop/dockview

Dockview integration with shell awareness.

- `BigtopDockview` — dockview wrapped with theme integration and event hooks
- `PanelRegistry` — typed panel registration via `createPanelRegistry()`
- `CustomTab` — default tab component with icons + close button
- `LayoutPersistence` — save/restore to pluggable storage backend (localStorage default)
- `theme.css` — dockview CSS variable overrides mapped to `--bigtop-*`

**Dependencies**: `@bigtop/core` (workspace), dockview-react (peer), react (peer)

### Dependency Graph

```
shadcn ← core ← dockview
```

## Tooling Packages

### @bigtop/typescript-config

Presets:
- `base.json` — strict, ESM, bundler resolution, ES2024 target, `noUncheckedIndexedAccess`, `verbatimModuleSyntax`
- `library.json` — extends base, emits declarations + declarationMap, `outDir: dist`
- `app.json` — extends base, `noEmit: true`
- `react-library.json` — extends library, adds JSX
- `react-app.json` — extends app, adds JSX

### @bigtop/eslint-config

ESLint 9 flat config:
- `base.js` — TypeScript strict rules, consistent-type-imports
- `react.js` — extends base + react-hooks plugin

## TypeScript Build Strategy

Bun runs TypeScript directly at dev time. `tsc` is only used for:
- Declaration file generation for published packages
- Type checking (`tsc --noEmit`)

Moon orchestrates build order via `deps: ["^:build"]`. No `tsc --build`, no project references.

## CSS Variable Theming

### Framework layer (`--bigtop-*`)

Defined in `@bigtop/core/theme/tokens.css`:

```css
:root {
  --bigtop-bg-base: var(--bigtop-theme-bg-base, #1e1e1e);
  --bigtop-bg-panel: var(--bigtop-theme-bg-panel, #252526);
  --bigtop-bg-input: var(--bigtop-theme-bg-input, #3c3c3c);
  --bigtop-bg-hover: var(--bigtop-theme-bg-hover, #2a2d2e);
  --bigtop-fg: var(--bigtop-theme-fg, #cccccc);
  --bigtop-fg-muted: var(--bigtop-theme-fg-muted, #858585);
  --bigtop-accent: var(--bigtop-theme-accent, #007fd4);
  --bigtop-border: var(--bigtop-theme-border, #3c3c3c);
}
```

Components use `--bigtop-*`. Theme presets override `--bigtop-theme-*`. Platform adapters (VS Code) map `--bigtop-theme-*` to host variables.

### Theme modes

`ThemeProvider` supports three modes:
- **dark** — dark preset variables
- **light** — light preset variables
- **system** — follows `prefers-color-scheme`, updates on OS change

Stored via `SettingsApi`. `data-theme` attribute on root element for CSS targeting.

### App consumption

Consumer apps (e.g., nenju) use `--bigtop-*` directly via Tailwind v4's `@theme` directive. App-specific tokens use their own namespace (`--nenju-*`) only if needed.

### shadcn integration

`@bigtop/shadcn` maps shadcn's default CSS variables to `--bigtop-*` so primitives inherit the shell theme automatically.

## CI/CD

### Phase 1 (now): PR checks

```yaml
# .github/workflows/ci.yml
name: CI
on: [pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: moonrepo/setup-toolchain@v0
      - run: moon ci
```

`moon ci` detects affected projects from the diff and runs typecheck + lint + test.

### Phase 2 (later): Publishing

Changesets for version management. CI runs `changeset version` on merge to main, opens release PR. Merging release PR triggers `changeset publish`.

## Moon Task Templates

### Library tasks (packages/*)

```yaml
language: 'typescript'
tags: ['package']
toolchain:
  default: 'bun'

tasks:
  build:
    command: 'tsc'
    inputs:
      - '@globs(sources)'
    outputs:
      - 'dist'
    deps:
      - '^:build'
  typecheck:
    command: 'tsc --noEmit'
    inputs:
      - '@globs(sources)'
  lint:
    command: 'eslint src/'
    inputs:
      - '@globs(sources)'
  test:
    command: 'bun test'
    inputs:
      - '@globs(sources)'
      - '@globs(tests)'
  clean:
    command: 'rm -rf dist'
```

### App tasks (sandbox/*)

```yaml
language: 'typescript'
tags: ['app']
toolchain:
  default: 'bun'

tasks:
  dev:
    command: 'vite dev'
    inputs:
      - '@globs(sources)'
    local: true
  build:
    command: 'vite build'
    inputs:
      - '@globs(sources)'
    outputs:
      - 'dist'
    deps:
      - '^:build'
  typecheck:
    command: 'tsc --noEmit'
    inputs:
      - '@globs(sources)'
  lint:
    command: 'eslint src/'
    inputs:
      - '@globs(sources)'
  clean:
    command: 'rm -rf dist'
```

## Build Phases

### Phase 1: Monorepo scaffold
- Moon config, bun workspaces, tooling packages
- Package stubs with build/typecheck/lint wired up
- `tmp/` gitignored, `sandbox/` ready
- GitHub Actions CI

### Phase 2: @bigtop/shadcn
- `shadcn init` with Tailwind v4
- Initial primitives: Button, Badge, Tooltip, DropdownMenu, Dialog, Input, ScrollArea
- CSS variables mapped to `--bigtop-*`

### Phase 3: @bigtop/core
Build order within the package:
1. ThemeProvider + tokens.css + dark/light/system presets
2. Shell + ActivityBar + StatusBar + TitleBar
3. CommandRegistry + CommandPalette (cmdk)
4. KeybindingManager (tinykeys) + useKeybinding
5. Notifications (sonner) + useNotify
6. Settings (SettingsApi, SettingsProvider, useSettings, LocalStorageSettingsApi)
7. PlatformApi interface (web defaults)

### Phase 4: @bigtop/dockview
1. BigtopDockview wrapper with theme integration
2. PanelRegistry + typed registration
3. CustomTab with icons + close
4. LayoutPersistence (pluggable storage, localStorage default)
5. theme.css (dockview CSS variable overrides)

### Phase 5: sandbox/nenju
Nenju web app on a branch, consuming @bigtop/core + @bigtop/dockview:
- BeadsApi interface + HttpBeadsApi + TanStack Query hooks
- Hono server + REST API + WebSocket change notifications
- Core components: IssuesList, KanbanBoard, BeadDetail, Dashboard

## Deferred Work

Tracked in beads:
- **bigtop-6ev** — Electron platform adapter spike (P3)
- **bigtop-n5m** — VS Code WebviewPanel adapter spike (P3)

Also deferred:
- Changeset publishing automation
- Example apps (added when API stabilizes)
- Documentation site
