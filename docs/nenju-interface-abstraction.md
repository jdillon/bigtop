# Nenju — Multi-Target from One Codebase

## Goal

Two concerns, two repos:

1. **Bigtop** (`@bigtop/*`) — Lightweight application shell framework. Provides
   the chrome that sits between a raw panel library and a full IDE: activity bar,
   status bar, command palette, keybindings, notifications, settings, theming.
   Built on dockview + cmdk + tinykeys + sonner. ~70kb deps + ~500 lines of shell.
   Targets: web, Electron, VS Code WebviewPanel.

2. **Nenju** (`@nenju/*`) — The beads issue tracker app, built on Bigtop.
   (Working name — 念珠, Japanese prayer beads.) Components, hooks, backend,
   API. Produces standalone web app, desktop app, and VS Code extension.

```
bigtop/                    # Application shell framework
  packages/
    core/                  # @bigtop/core — shell, theming, settings, commands
    dockview/              # @bigtop/dockview — dockview integration + layout persistence
    electron/              # @bigtop/electron — Electron main process helpers
    vscode/                # @bigtop/vscode — WebviewPanel shell adapter

nenju/                     # Beads app (depends on @bigtop/*)
  packages/
    core/                  # @nenju/core — components, hooks, BeadsApi, backend
    web/                   # @nenju/web — Hono server + HttpBeadsApi
    desktop/               # @nenju/desktop — Electron app
    vscode/                # @nenju/vscode — VS Code extension
```

> Could start as one monorepo and split bigtop out once it stabilizes.

## Tech Stack

| Layer | Choice | Notes |
|-------|--------|-------|
| Build system | Turborepo | Manages package graph + caching |
| UI framework | React 18+ | Shared across all targets |
| Styling | Tailwind v4 | Utility classes, CSS variable theming |
| Component library | shadcn/ui (Radix primitives) | Buttons, badges, inputs, tooltips, dropdowns |
| Layout | dockview (via @bigtop/dockview) | Tabs, splits, floating panels, persistence |
| Command palette | cmdk (via @bigtop/core) | Keyboard-driven command execution |
| Keyboard shortcuts | tinykeys (via @bigtop/core) | Lightweight keybinding (~400B) |
| Notifications | sonner (via @bigtop/core) | Toast notifications |
| Icons | Lucide React | Consistent across all targets |
| Data fetching | TanStack Query | Caching, optimistic updates, background refetch |
| Routing | TanStack Router | URL-based views (`/issues`, `/kanban`, `/bead/:id`) |
| Server (web) | Hono | Lightweight, WebSocket support |
| Desktop shell | Electron | BrowserWindow + IPC |
| VS Code | Single WebviewPanel | Full app in one editor tab. Not native sidebar. |

## Repo Structure

### Bigtop (application shell framework)

```
bigtop/
  turbo.json
  package.json

  packages/
    core/                              # @bigtop/core
      src/
        shell/
          ActivityBar.tsx              # Icon sidebar with tooltips
          StatusBar.tsx                # Fixed bottom bar
          TitleBar.tsx                 # Optional top bar
          Shell.tsx                    # Composes all shell chrome
        commands/
          CommandPalette.tsx           # cmdk-based command palette
          CommandRegistry.ts           # Register/execute commands
          useCommand.ts                # Hook to register commands from components
        keybindings/
          KeybindingManager.ts         # tinykeys wrapper
          useKeybinding.ts             # Hook to bind keys from components
        notifications/
          Toaster.tsx                  # sonner wrapper
          useNotify.ts                 # Hook: notify({ title, type })
        settings/
          SettingsProvider.tsx          # React context for settings
          SettingsApi.ts               # Interface: get/set/onChange
          useSettings.ts               # Hook: useSettings("key", default)
        theme/
          ThemeProvider.tsx             # CSS variable injection
          tokens.css                   # --bigtop-* variable definitions
        types.ts                       # Shared types

    dockview/                          # @bigtop/dockview
      src/
        BigtopDockview.tsx             # Dockview + shell integration
        PanelRegistry.ts              # Register panel types
        LayoutPersistence.ts           # Save/restore layout state
        CustomTab.tsx                  # Default tab component with icons
        theme.css                      # Dockview CSS variable overrides

    electron/                          # @bigtop/electron
      src/
        createElectronShell.ts         # Main process helpers
        preload.ts                     # IPC bridge template
        ElectronSettingsApi.ts         # Settings via electron-store
        ElectronPlatform.ts            # Platform services (clipboard, file open, etc.)

    vscode/                            # @bigtop/vscode
      src/
        createVscodeShell.ts           # WebviewPanel creation + message handling
        VscodeSettingsApi.ts           # Settings via vscode.workspace.getConfiguration
        VscodePlatform.ts             # Platform services via postMessage
        theme.css                      # Maps --bigtop-* to --vscode-*
```

### Nenju (app, depends on @bigtop/*)

```
nenju/
  turbo.json
  package.json

  packages/
    core/                              # @nenju/core
      src/
        backend/
          BeadsBackend.ts              # Interface (pure)
          BeadsDoltBackend.ts          # Dolt/SQL (pure Node.js)
          BeadsCommandRunner.ts        # CLI runner (pure Node.js)
        types/
          bead.ts                      # Bead, BeadStatus, BeadPriority
          normalization.ts             # Status/priority normalization
        components/
          IssuesList.tsx               # Issue list table with filter/sort
          KanbanBoard.tsx              # Kanban columns
          BeadDetail.tsx               # Single bead detail view
          Dashboard.tsx                # Summary stats
          common/                      # StatusDot, Badge, LabelBadge, etc.
        hooks/
          useBeads.ts                  # TanStack Query: list beads
          useBead.ts                   # TanStack Query: single bead
          useMutateBead.ts             # TanStack Query: update/create/close
        api/
          BeadsApi.ts                  # Abstract data access interface
        panels.ts                      # Register beads panels with @bigtop/dockview
        commands.ts                    # Register beads commands with @bigtop/core

    web/                               # @nenju/web — standalone
      src/
        server.ts                      # Hono + WebSocket + BeadsDoltBackend
        api-routes.ts                  # REST endpoints
        main.tsx                       # Vite entry: BigtopShell + BeadsApi
        HttpBeadsApi.ts                # fetch-based BeadsApi
        cli.ts                         # CLI: `beads-web`

    desktop/                           # @nenju/desktop — Electron
      src/
        main.ts                        # Electron main process
        renderer/main.tsx              # Entry: BigtopShell + IpcBeadsApi
        IpcBeadsApi.ts                 # IPC-based BeadsApi

    vscode/                            # @nenju/vscode — extension
      src/
        extension.ts                   # WebviewPanel + message handler
        VscodeBeadsApi.ts              # postMessage-based BeadsApi
        webview/index.tsx              # Entry: BigtopShell + VscodeBeadsApi
```

## Key Insight: Single WebviewPanel for VS Code

Instead of multiple native VS Code sidebar panels (which would require separate
UI), we use a **single WebviewPanel** that opens as an editor tab. The full
React app — dockview, Tailwind, shadcn, TanStack — runs inside it, identical to
the standalone and desktop versions.

```typescript
// packages/vscode/src/extension.ts
export function activate(context: vscode.ExtensionContext) {
  const backend = new BeadsDoltBackend({ ... });

  context.subscriptions.push(
    vscode.commands.registerCommand("beads.open", () => {
      const panel = vscode.window.createWebviewPanel(
        "beads", "Beads", vscode.ViewColumn.One,
        { enableScripts: true, retainContextWhenHidden: true }
      );
      panel.webview.html = getHtml(panel.webview, context.extensionUri);

      // Handle messages from webview (BeadsApi calls)
      panel.webview.onDidReceiveMessage(async (msg) => {
        const result = await handleApiCall(backend, msg);
        panel.webview.postMessage({ id: msg.id, result });
      });
    })
  );
}
```

The VS Code package is ~150 lines of glue. No UI code at all.

**Trade-off**: Loses the "integrated sidebar" feel (panels beside Explorer,
Source Control). Beads opens as a tab in the editor area instead. But this means
zero UI duplication and more screen space for the app.

## The Compatibility Layer: `BeadsApi`

Typed interface that TanStack Query hooks call. Each target provides its own
implementation.

```typescript
// packages/core/src/api/BeadsApi.ts

export interface BeadsApi {
  listBeads(filters?: BeadFilters): Promise<Bead[]>;
  getBead(id: string): Promise<Bead>;
  updateBead(id: string, updates: Partial<Bead>): Promise<Bead>;
  createBead(data: CreateBeadArgs): Promise<Bead>;
  closeBead(id: string, reason?: string): Promise<Bead>;
  listComments(beadId: string): Promise<BeadComment[]>;
  addComment(beadId: string, text: string): Promise<void>;
  addDependency(from: string, to: string, type: string): Promise<void>;
  removeDependency(from: string, to: string): Promise<void>;
  getProject(): Promise<BeadsProject | null>;
  getSummary(): Promise<BeadsSummary>;
  onDataChanged(handler: () => void): () => void;
}
```

### Implementations

```
┌──────────────────────────────────────────────────────┐
│  packages/core                                       │
│                                                      │
│  App.tsx (dockview + providers)                      │
│    └── Components → TanStack Query hooks             │
│                          │                           │
│                          ▼                           │
│                    BeadsApi interface                 │
│                          │                           │
└──────────────────────────┼───────────────────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
         ▼                 ▼                 ▼
  HttpBeadsApi      IpcBeadsApi      VscodeBeadsApi
  (packages/web)    (packages/desktop) (packages/vscode)
  fetch → Hono      ipcRenderer       postMessage
  WS for changes    IPC for changes   messages for changes
```

### TanStack Query Hooks (shared)

```typescript
// packages/core/src/hooks/useBeads.ts
export function useBeads(filters?: BeadFilters) {
  const api = useBeadsApi(); // React context
  return useQuery({
    queryKey: ["beads", filters],
    queryFn: () => api.listBeads(filters),
  });
}

// packages/core/src/hooks/useMutateBead.ts
export function useUpdateBead() {
  const api = useBeadsApi();
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ id, updates }) => api.updateBead(id, updates),
    onSuccess: () => qc.invalidateQueries({ queryKey: ["beads"] }),
  });
}
```

## Styling

Shared components use `--beads-*` CSS variables with Tailwind:

```css
/* packages/core/src/theme.css */
@theme {
  --color-bg-base: var(--beads-bg-base);
  --color-bg-panel: var(--beads-bg-panel);
  --color-bg-input: var(--beads-bg-input);
  --color-bg-hover: var(--beads-bg-hover);
  --color-fg: var(--beads-fg);
  --color-fg-muted: var(--beads-fg-muted);
  --color-accent: var(--beads-accent);
  --color-border: var(--beads-border);
}
```

Each target defines the actual values:

**Web/Desktop** — hardcoded Dark+ palette:
```css
:root {
  --beads-bg-base: #1e1e1e;
  --beads-bg-panel: #252526;
  --beads-fg: #cccccc;
  --beads-accent: #007fd4;
  /* etc */
}
```

**VS Code** — maps to host theme:
```css
:root {
  --beads-bg-base: var(--vscode-editor-background);
  --beads-bg-panel: var(--vscode-sideBar-background);
  --beads-fg: var(--vscode-editor-foreground);
  --beads-accent: var(--vscode-focusBorder);
  /* etc */
}
```

VS Code users get their chosen theme. Standalone/desktop get Dark+ defaults.

## Data Flow Per Target

### Web (Hono + WebSocket)
```
Component → TanStack Query → HttpBeadsApi → fetch() → Hono API → BeadsDoltBackend
Changes: server polls dolt_hashof_db() → WS push → invalidateQueries()
```

### Desktop (Electron IPC)
```
Component → TanStack Query → IpcBeadsApi → ipcRenderer.invoke() → main process → BeadsDoltBackend
Changes: main process polls → ipcMain.send() → invalidateQueries()
```

### VS Code (postMessage)
```
Component → TanStack Query → VscodeBeadsApi → postMessage → extension host → BeadsDoltBackend
Changes: ProjectManager.onDataChanged → postMessage → invalidateQueries()
```

## Cross-Cutting Concerns

All handled by Bigtop with platform-specific adapters:

### Settings
- **@bigtop/core**: `SettingsApi` interface + `useSettings()` hook + `SettingsProvider`
- **Web**: `LocalStorageSettingsApi` (or JSON config file)
- **Desktop**: `ElectronSettingsApi` (electron-store)
- **VS Code**: `VscodeSettingsApi` (vscode.workspace.getConfiguration)

### Notifications / Toasts
- **@bigtop/core**: `useNotify()` hook wrapping sonner
- All targets use in-app toasts via Bigtop. No platform bridging needed.

### Command Palette + Keybindings
- **@bigtop/core**: `CommandRegistry` + `CommandPalette` (cmdk) + keybindings (tinykeys)
- Apps register commands: `registry.register({ id: "beads.refresh", label: "Refresh", handler: ... })`
- Keybindings bound to command IDs: `{ "Cmd+R": "beads.refresh" }`
- Works identically across all targets (it's all in-browser)

### Clipboard
- **Web/Desktop**: `navigator.clipboard.writeText()` (works in Electron too)
- **VS Code**: `navigator.clipboard` works in WebviewPanels

### File Opening
- **@bigtop/core**: `PlatformApi` interface with `openFile(path, line?)`
- **Web**: No-op or link to git host
- **Desktop**: `shell.openPath()` via Electron IPC
- **VS Code**: `vscode.workspace.openTextDocument()` via postMessage

## Build Commands

```bash
turbo build                            # All packages
turbo build --filter=@nenju/web     # Standalone only
turbo build --filter=@nenju/vscode  # VS Code extension only
turbo dev --filter=@nenju/web       # Standalone dev (Vite HMR)

cd packages/desktop
bun run dev                            # Electron dev mode
bun run package                        # Build distributable
```

## Migration Path

### Phase 1: Monorepo + Core
- Turborepo setup
- Extract backend + types to packages/core
- Define `BeadsApi` interface + TanStack Query hooks
- Core React components with Tailwind + shadcn + dockview
- `--beads-*` CSS variable system

### Phase 2: Standalone web
- packages/web — Hono server, REST API, WebSocket, HttpBeadsApi
- Dockview layout, Dark+ theme
- CLI: `beads-web`

### Phase 3: VS Code (single WebviewPanel)
- packages/vscode — thin shell, VscodeBeadsApi
- Same App.tsx, same dockview, same components
- Maps --beads-* to --vscode-*

### Phase 4: Desktop
- packages/desktop — Electron shell, IpcBeadsApi
- Same App.tsx as web
- Package with electron-builder

### Phase 5: Polish
- TanStack Router for URL navigation
- Optimistic updates
- Settings UI
- Light theme variant
- Layout presets
- Keyboard shortcuts

## Decisions

- **Bigtop** (`@bigtop/*`): Build a lightweight application shell framework.
  Provides activity bar, status bar, command palette (cmdk), keybindings
  (tinykeys), notifications (sonner), settings, theming. Wraps dockview for
  panel layout. ~70kb deps + ~500 lines of shell code. Reusable across projects.
- **Single WebviewPanel for VS Code**: Full app in one editor tab. Zero UI
  duplication across targets. Same dockview + Bigtop shell everywhere.
- **`BeadsApi` interface**: Typed data access. Each target provides HTTP, IPC,
  or postMessage implementation. Components never know their host.
- **TanStack Query**: Shared hooks. Transport-agnostic. `onDataChanged`
  invalidates cache for real-time updates.
- **Tailwind + shadcn + CSS variable theming**: Bigtop defines `--bigtop-*`
  variables. Apps extend with their own tokens. VS Code maps to `--vscode-*`.
- **Turborepo**: Build graph + caching. `workspace:*` deps.
- **Name: Bigtop** — the tent that houses the whole show. `@bigtop/core`,
  `@bigtop/dockview`, `@bigtop/electron`, `@bigtop/vscode`. npm: available.

## What Bigtop Provides (vs. what the app provides)

| Concern | Bigtop | App (Nenju) |
|---------|--------|----------------|
| Activity bar | Shell component + API | Icons, panel bindings |
| Status bar | Shell component + API | Content (branch, Dolt status) |
| Panel layout | Dockview integration | Panel components + registration |
| Command palette | cmdk + CommandRegistry | Command definitions |
| Keyboard shortcuts | tinykeys + binding API | Key mappings |
| Notifications | sonner + useNotify() | When to notify |
| Settings | Provider + useSettings() | Setting keys + defaults |
| Theming | CSS variables + ThemeProvider | Theme tokens |
| Data layer | — | BeadsApi + TanStack Query hooks |
| Components | — | IssuesList, Kanban, Detail, etc. |

## Showcase

Dockview + Tailwind + shadcn prototype: `sandbox/dockview-showcase/`
- VS Code Dark+ theme with activity bar, status bar, custom tabs
- Issues list, Kanban board, Dashboard panels
- Click-to-split detail panels, floating panels
- Run: `cd sandbox/dockview-showcase && bun run dev`

## Open Questions

- **Mono vs. multi repo**: Start with one monorepo containing both bigtop and
  beads-ui packages, split bigtop out when it stabilizes? Or separate from day 1?
- **Port selection** (web): Fixed vs. deterministic from project path.
- **CLI distribution**: `bin` in @nenju/web, or ship with `bd` CLI.
- **Electron packaging**: Homebrew cask? DMG?
- **GitHub org**: Create a `bigtop` org for the framework packages?
