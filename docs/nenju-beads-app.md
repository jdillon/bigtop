# Nenju — Visual Interface for Beads

> Working name. 念珠 = Japanese prayer beads.

## What Is This

Nenju is a standalone web/desktop application for managing
[Beads](https://github.com/steveyegge/beads) issues. It replaces the VS Code
extension as the primary interface, running in a browser tab (e.g., inside
Superset) or as a native Electron app.

Built on the Bigtop application shell framework. Uses dockview for layout,
TanStack Query for data, Tailwind + shadcn for UI.

**npm scope**: `@nenju/*`

## Why

The VS Code extension works but Jason mostly opens VS Code just to look at
beads. A browser tab in Superset is more natural. The extension can come back
later as a thin wrapper — the same React app running inside a single
WebviewPanel.

## Architecture

```
@nenju/core          @nenju/web          @nenju/desktop
(components,     →   (Hono server,   →   (Electron,
 hooks,               HttpBeadsApi,       IpcBeadsApi)
 BeadsApi)            WebSocket)

      ↑ depends on
  @bigtop/core + @bigtop/dockview
```

### Package Structure

```
nenju/
  turbo.json
  package.json

  packages/
    core/                          # @nenju/core
      src/
        backend/
          BeadsBackend.ts          # Interface (pure, from vscode-beads)
          BeadsDoltBackend.ts      # Dolt/SQL implementation (pure Node.js)
          BeadsCommandRunner.ts    # bd CLI runner (pure Node.js)
        types/
          bead.ts                  # Bead, BeadStatus, BeadPriority
          normalization.ts         # normalizeStatus, normalizePriority
        components/
          IssuesList.tsx           # Table view with filter, sort, column state
          KanbanBoard.tsx          # Kanban columns by status
          BeadDetail.tsx           # Single bead: metadata, description, comments
          Dashboard.tsx            # Summary stats, priority breakdown, activity
          common/
            StatusDot.tsx
            StatusBadge.tsx
            PriorityBadge.tsx
            TypeBadge.tsx
            LabelBadge.tsx
            FilterChip.tsx
            Markdown.tsx           # Rendered markdown for descriptions/notes
        hooks/
          useBeads.ts              # TanStack Query: list with filters
          useBead.ts               # TanStack Query: single bead by ID
          useUpdateBead.ts         # Mutation: update fields
          useCreateBead.ts         # Mutation: create new bead
          useCloseBead.ts          # Mutation: close with reason
          useComments.ts           # Query + mutation for comments
          useDependencies.ts       # Mutation: add/remove deps
          useSummary.ts            # Query: dashboard summary
        api/
          BeadsApi.ts              # Abstract data access interface
          BeadsApiProvider.tsx     # React context provider
          useBeadsApi.ts           # Hook to access BeadsApi from context
        panels.ts                  # Panel registration for @bigtop/dockview
        commands.ts                # Command registration for @bigtop/core
        default-layout.ts          # Default dockview layout config

    web/                           # @nenju/web — standalone
      src/
        server.ts                  # Hono + static files + WebSocket
        api-routes.ts              # REST: /api/beads, /api/beads/:id, etc.
        ws-handler.ts              # WebSocket: change notifications
        HttpBeadsApi.ts            # BeadsApi impl using fetch()
        main.tsx                   # Vite entry point
        App.tsx                    # Mounts Bigtop Shell + providers
        theme.css                  # --bigtop-theme-* = Dark+ defaults
        cli.ts                     # CLI entry: `nenju` or `nenju serve`
      vite.config.ts
      package.json                 # bin: { "nenju": "./dist/cli.js" }

    desktop/                       # @nenju/desktop — Electron
      src/
        main.ts                    # Electron main process
        preload.ts                 # IPC bridge
        ipc-handlers.ts            # beads:list, beads:show, etc.
        IpcBeadsApi.ts             # BeadsApi impl using ipcRenderer
        renderer/
          main.tsx                 # Entry, mounts same App
          theme.css
      electron-builder.yml
```

## Data Access: BeadsApi

The core abstraction. All components talk through this interface via TanStack
Query hooks. Each target provides an implementation.

```typescript
// @nenju/core/src/api/BeadsApi.ts

export interface BeadsApi {
  // Queries
  listBeads(filters?: BeadFilters): Promise<Bead[]>;
  getBead(id: string): Promise<Bead>;
  getSummary(): Promise<BeadsSummary>;
  listComments(beadId: string): Promise<BeadComment[]>;

  // Mutations
  createBead(args: CreateBeadArgs): Promise<Bead>;
  updateBead(id: string, updates: Partial<Bead>): Promise<Bead>;
  closeBead(id: string, reason?: string): Promise<Bead>;
  addComment(beadId: string, text: string): Promise<void>;
  addDependency(from: string, to: string, type: string): Promise<void>;
  removeDependency(from: string, to: string): Promise<void>;

  // Real-time
  onDataChanged(handler: () => void): () => void;
}
```

### Implementations

**HttpBeadsApi** (`@nenju/web`):
```typescript
class HttpBeadsApi implements BeadsApi {
  private ws: WebSocket;

  async listBeads(filters) {
    const params = new URLSearchParams(filters);
    const res = await fetch(`/api/beads?${params}`);
    return res.json();
  }

  onDataChanged(handler) {
    this.ws.addEventListener("message", (e) => {
      if (JSON.parse(e.data).type === "data-changed") handler();
    });
    return () => { /* cleanup */ };
  }
}
```

**IpcBeadsApi** (`@nenju/desktop`):
```typescript
class IpcBeadsApi implements BeadsApi {
  async listBeads(filters) {
    return ipcRenderer.invoke("beads:list", filters);
  }

  onDataChanged(handler) {
    ipcRenderer.on("beads:data-changed", handler);
    return () => ipcRenderer.removeListener("beads:data-changed", handler);
  }
}
```

### TanStack Query Hooks

```typescript
// @nenju/core/src/hooks/useBeads.ts
export function useBeads(filters?: BeadFilters) {
  const api = useBeadsApi();
  return useQuery({
    queryKey: ["beads", filters],
    queryFn: () => api.listBeads(filters),
  });
}

// Change notifications invalidate the cache
// Set up in App.tsx:
const api = useBeadsApi();
useEffect(() => {
  return api.onDataChanged(() => {
    queryClient.invalidateQueries({ queryKey: ["beads"] });
  });
}, [api]);
```

## Web Server (Hono)

Single-project, CWD-scoped. Auto-starts Dolt via `BeadsDoltBackend.ensureServerRunning()`.

```bash
cd ~/my-project
nenju              # starts server, opens http://localhost:3456
```

### REST API

```
GET    /api/beads              # list (query params for filters)
GET    /api/beads/:id          # show single bead
POST   /api/beads              # create
PATCH  /api/beads/:id          # update
POST   /api/beads/:id/close    # close
GET    /api/beads/:id/comments # list comments
POST   /api/beads/:id/comments # add comment
POST   /api/beads/:id/deps     # add dependency
DELETE /api/beads/:id/deps/:to # remove dependency
GET    /api/summary            # dashboard summary
WS     /ws                     # change notifications
```

### WebSocket

Server polls `dolt_hashof_db()` on interval. When token changes, broadcasts
`{ type: "data-changed" }` to all connected clients. Clients invalidate
TanStack Query cache.

## Views / Panels

### Issues List
- Table with columns: status, ID, title, priority, assignee, labels, updated
- Filter bar: text search, status chips, priority, type, assignee
- Sort by any column
- Click row → opens detail in split panel or new tab
- Float button → opens detail in floating panel
- Inline status/priority dropdowns for quick updates

### Kanban Board
- Columns: Open, In Progress, Blocked, Closed
- Cards show title, ID, priority, labels, assignee avatar
- Click card → opens detail
- Future: drag to change status

### Dashboard
- Stats cards: total, open, in progress, blocked, closed
- Priority breakdown bars
- Recent activity list

### Bead Detail
- Header: ID, status badge, priority badge, type badge
- Metadata grid: assignee, updated, labels
- Sections: description, design notes, acceptance criteria, working notes
- Dependencies: depends-on and blocks lists with status
- Comments with add comment form
- Actions: update status, assign, add labels, close

## What Comes From vscode-beads

These files move to @nenju/core essentially unchanged:

| From vscode-beads | To @nenju/core | Notes |
|-------------------|----------------|-------|
| `src/backend/BeadsBackend.ts` | `backend/BeadsBackend.ts` | Pure interface, no changes |
| `src/backend/BeadsDoltBackend.ts` | `backend/BeadsDoltBackend.ts` | Pure Node.js, no changes |
| `src/backend/BeadsCommandRunner.ts` | `backend/BeadsCommandRunner.ts` | Pure Node.js, no changes |
| `src/backend/types.ts` | `types/` | Split into bead.ts + normalization.ts |
| `src/webview/views/IssuesView.tsx` | `components/IssuesList.tsx` | Rewrite with Tailwind + shadcn |
| `src/webview/views/KanbanBoard.tsx` | `components/KanbanBoard.tsx` | Rewrite with Tailwind + shadcn |
| `src/webview/views/DetailsView.tsx` | `components/BeadDetail.tsx` | Rewrite with Tailwind + shadcn |
| `src/webview/views/DashboardView.tsx` | `components/Dashboard.tsx` | Rewrite with Tailwind + shadcn |
| `src/webview/common/*` | `components/common/*` | Rewrite with shadcn components |
| `src/webview/hooks/*` | `hooks/*` | Replace with TanStack Query |

Backend files are copy-paste. UI components are rewritten with modern stack
(Tailwind, shadcn, TanStack Query) but same layout and behavior.

## Superset Integration

Superset is an Electron app with browser panes (`<webview>` tags).

1. Run `nenju` in a project directory
2. Open `http://localhost:3456` in a Superset browser pane
3. URL persists in Superset workspace state

No plugin needed. Just a URL in a browser pane.

## Future: VS Code Extension

When ready to bring back VS Code support, it's a thin package:

```
nenju/
  packages/
    vscode/                    # @nenju/vscode
      src/
        extension.ts           # Creates single WebviewPanel
        VscodeBeadsApi.ts      # postMessage-based BeadsApi
        webview/index.tsx      # Mounts same App from @nenju/core
        theme.css              # Maps --bigtop-* to --vscode-*
```

~150 lines of extension glue. The full Bigtop + dockview app runs inside
a single WebviewPanel. No native sidebar panels, no UI duplication.

## Build & Dev

```bash
turbo build                        # All packages
turbo dev --filter=@nenju/web      # Standalone dev (Vite HMR + server)

cd packages/desktop
bun run dev                        # Electron dev mode
bun run package                    # Build distributable
```

## Prototype

Working dockview + Tailwind + shadcn prototype:
`vscode-beads/sandbox/dockview-showcase/`

Run: `cd sandbox/dockview-showcase && bun run dev` → http://localhost:5174

Features: VS Code Dark+ theme, activity bar, status bar, custom tab renderer
with icons, issues list, kanban board, dashboard, click-to-split detail panels,
floating panels, layout persistence.
