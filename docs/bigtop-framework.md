# Bigtop — Lightweight Application Shell Framework

## What Is This

Bigtop is a lightweight framework for building panel-based desktop/web
applications with an IDE-like chrome — without being an IDE. It fills the gap
between raw panel libraries (dockview) and full IDE frameworks (Theia, Molecule)
that ship megabytes of editor code you don't need.

**npm scope**: `@bigtop/*`
**Domains**: bigtop.dev (available as of 2026-03-23)

## The Problem

Building a panel-based app today means choosing between:

1. **Just a layout library** (dockview, allotment) — handles panels but you
   build everything else: activity bar, status bar, command palette, keybindings,
   notifications, settings, theming. ~500 lines minimum.

2. **Full IDE framework** (Theia, Molecule, OpenSumi) — has the shell but is
   tightly coupled to Monaco editor. Ships 3-5MB+ you don't need. Fighting the
   framework to not be an IDE.

3. **Custom from scratch** — what OpenCode desktop did: ~5,400 lines of layout
   system, tabs, persistence, resize handles. Works but significant to maintain.

Bigtop is option 4: the shell chrome as a focused framework. ~70kb of deps,
~500 lines of shell code. Batteries included, editor not included.

## Tech Stack

| Concern | Library | Size |
|---------|---------|------|
| Panel layout | dockview | 57kb, zero deps |
| Command palette | cmdk | ~4kb |
| Keyboard shortcuts | tinykeys | ~400B |
| Notifications/toasts | sonner | ~5kb |
| UI primitives | React 18+ | peer dep |
| Styling | Tailwind v4 | build-time |
| Component primitives | shadcn/ui (Radix) | copy-paste, no dep |
| Icons | Lucide React | tree-shakeable |

## Package Structure

```
bigtop/
  turbo.json
  package.json

  packages/
    core/                          # @bigtop/core
      src/
        shell/
          Shell.tsx                # Root shell component, composes all chrome
          ActivityBar.tsx          # Left icon sidebar with tooltips
          StatusBar.tsx            # Bottom bar with sections
          TitleBar.tsx             # Optional top bar
        commands/
          CommandPalette.tsx       # cmdk-based, Cmd+K to open
          CommandRegistry.ts      # Register/execute/list commands
          useCommand.ts           # Hook: register from components
        keybindings/
          KeybindingManager.ts    # tinykeys wrapper
          useKeybinding.ts        # Hook: bind keys from components
        notifications/
          Toaster.tsx             # sonner wrapper with theming
          useNotify.ts            # Hook: success/error/info toasts
        settings/
          SettingsProvider.tsx     # React context
          SettingsApi.ts          # Interface: get/set/onChange
          useSettings.ts          # Hook: typed settings access
        theme/
          ThemeProvider.tsx        # Injects CSS variables
          tokens.css              # --bigtop-* variable definitions
          presets/
            dark.css              # VS Code Dark+ defaults
            light.css             # Light theme
        types.ts

    dockview/                      # @bigtop/dockview
      src/
        BigtopDockview.tsx         # Dockview wrapped with shell integration
        PanelRegistry.ts          # Typed panel registration
        LayoutPersistence.ts      # Save/restore to storage backend
        CustomTab.tsx             # Default tab with icons + close
        theme.css                 # Dockview CSS variable overrides

    electron/                      # @bigtop/electron
      src/
        createElectronShell.ts    # Main process boilerplate
        preload.ts                # IPC bridge template
        ElectronSettingsApi.ts    # electron-store adapter
        ElectronPlatform.ts       # Platform services (file open, clipboard)

    vscode/                        # @bigtop/vscode
      src/
        createVscodeShell.ts      # WebviewPanel creation + message loop
        VscodeSettingsApi.ts      # workspace.getConfiguration adapter
        VscodePlatform.ts         # postMessage-based platform services
        theme.css                 # Maps --bigtop-* to --vscode-*
```

## Core Concepts

### Shell

The `<Shell>` component composes all the chrome:

```tsx
import { Shell, ActivityBar, StatusBar } from "@bigtop/core";
import { BigtopDockview } from "@bigtop/dockview";

function App() {
  return (
    <Shell>
      <Shell.TitleBar>My App</Shell.TitleBar>
      <Shell.Content>
        <Shell.ActivityBar>
          <ActivityBar.Item icon={List} label="Issues" panelId="issues" />
          <ActivityBar.Item icon={Settings} label="Settings" position="bottom" />
        </Shell.ActivityBar>
        <BigtopDockview
          panels={panels}
          defaultLayout={defaultLayout}
          persistKey="my-app"
        />
      </Shell.Content>
      <Shell.StatusBar>
        <StatusBar.Section side="left">
          <StatusBar.Item>main</StatusBar.Item>
        </StatusBar.Section>
      </Shell.StatusBar>
    </Shell>
  );
}
```

### Commands

```typescript
import { useCommandRegistry } from "@bigtop/core";

// Register
const registry = useCommandRegistry();
registry.register({
  id: "app.refresh",
  label: "Refresh Data",
  keybinding: "Cmd+R",
  handler: () => queryClient.invalidateQueries(),
});

// The command palette (Cmd+K) automatically lists all registered commands.
```

### Settings

```typescript
import { useSettings } from "@bigtop/core";

// In a component
const [theme, setTheme] = useSettings("appearance.theme", "dark");
const [autoRefresh] = useSettings("data.autoRefresh", true);
```

Settings storage is pluggable:
- `LocalStorageSettingsApi` (web, default)
- `ElectronSettingsApi` (desktop, electron-store)
- `VscodeSettingsApi` (VS Code, workspace.getConfiguration)

### Theming

CSS custom properties with a provider:

```css
/* @bigtop/core/theme/tokens.css */
:root {
  --bigtop-bg-base: var(--bigtop-theme-bg-base, #1e1e1e);
  --bigtop-bg-panel: var(--bigtop-theme-bg-panel, #252526);
  --bigtop-bg-input: var(--bigtop-theme-bg-input, #3c3c3c);
  --bigtop-fg: var(--bigtop-theme-fg, #cccccc);
  --bigtop-fg-muted: var(--bigtop-theme-fg-muted, #858585);
  --bigtop-accent: var(--bigtop-theme-accent, #007fd4);
  --bigtop-border: var(--bigtop-theme-border, #3c3c3c);
}
```

Apps use `bg-[var(--bigtop-bg-base)]` etc. in Tailwind. Theme presets override
the `--bigtop-theme-*` layer. VS Code adapter maps to `--vscode-*`.

### Panel Registration

```typescript
import { createPanelRegistry } from "@bigtop/dockview";

const panels = createPanelRegistry({
  issues: {
    component: IssuesList,
    title: "Issues",
    icon: List,
  },
  detail: {
    component: BeadDetail,
    title: "Detail",
    icon: FileText,
    // Supports dynamic titles via params
    getTitle: (params) => params.beadId,
  },
});
```

### Layout Persistence

```typescript
<BigtopDockview
  panels={panels}
  defaultLayout={defaultLayout}
  persistKey="my-app-layout"     // localStorage key
  // or
  persistApi={electronStore}     // custom storage backend
/>
```

Layout serialized/restored automatically on mount/unmount and on changes.

## Platform Adapters

Each target provides platform-specific capabilities:

```typescript
interface PlatformApi {
  openFile(path: string, line?: number): Promise<void>;
  copyToClipboard(text: string): Promise<void>;
  openExternal(url: string): Promise<void>;
  getSettingsApi(): SettingsApi;
}
```

| Method | Web | Electron | VS Code |
|--------|-----|----------|---------|
| openFile | no-op / git link | shell.openPath | vscode.openTextDocument |
| copyToClipboard | navigator.clipboard | navigator.clipboard | navigator.clipboard |
| openExternal | window.open | shell.openExternal | vscode.env.openExternal |
| getSettingsApi | localStorage | electron-store | workspace.getConfiguration |

## What Bigtop Does NOT Do

- **No code editor**. No Monaco, no CodeMirror.
- **No file system**. No file tree, no file watching.
- **No terminal**. No xterm.js, no PTY.
- **No extension system**. Apps compose directly, no dynamic loading.
- **No framework lock-in**. React is a peer dep. Components are composable,
  not a monolithic shell you can't escape.

## Design Principles

1. **Compose, don't configure.** The shell is React components, not a config
   object. You arrange them how you want.
2. **Zero opinion on data.** Bigtop doesn't know about your API, your state
   management, or your data model. It provides chrome, not content.
3. **Escape hatches everywhere.** Don't want the activity bar? Don't render it.
   Want a custom tab renderer? Pass one. Every piece is optional.
4. **Light deps.** The entire framework is ~70kb. No tree you can't shake.

## Research & Prior Art

Evaluated before building:
- **Molecule** (DTStack) — closest match but Monaco is a hard dep. MIT.
- **Theia** (@theia/core) — most mature but massive. EPL-2.0. Can't strip editor.
- **OpenSumi** — similar to Theia, Chinese ecosystem.
- **Lumino** (JupyterLab) — has commands + keybindings but not React.
- **Piral** — micro-frontend container, wrong level of abstraction.

Full research: see `Application Shell Frameworks for IDE-like Web Apps.md`
in Jason's Obsidian vault.

## Prototype

A working dockview + Tailwind + shadcn prototype exists at:
`vscode-beads/sandbox/dockview-showcase/`

Features VS Code Dark+ theme, activity bar, status bar, custom tabs, issues
list, kanban board, dashboard, floating panels. Run with `bun run dev`.

## Build System

Turborepo. Each package has its own tsconfig and build step.

```bash
turbo build                        # All packages
turbo build --filter=@bigtop/core  # Just core
turbo dev                          # Dev mode
```
