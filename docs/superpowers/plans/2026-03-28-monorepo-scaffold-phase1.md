# Monorepo Scaffold — Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Scaffold the bigtop monorepo with shared tooling configs, package stubs, CI, and beads epic tracking.

**Architecture:** Bun workspaces + moon task orchestration. Shared TypeScript and ESLint configs in `tooling/`. Library package stubs in `packages/`. Moon manages all builds via `tsc` (declarations) and `tsc --noEmit` (type checking). No `tsc --build`, no project references for compilation.

**Tech Stack:** TypeScript 6.x, ESLint 10.x, typescript-eslint 8.x, Bun 1.3.11, moon 2.1.3

**Source spec:** `docs/2026-03-28-monorepo-scaffold-design.md`

---

### Task 1: Beads Epic + Stories

Create the overall bigtop implementation epic with phase stories.

- [ ] **Step 1: Create epic**

```bash
bd create --title="Bigtop implementation" --description="Epic tracking all bigtop framework implementation phases. See docs/2026-03-28-monorepo-scaffold-design.md for full spec." --type=epic --priority=1
```

- [ ] **Step 2: Create phase stories**

```bash
bd create --title="Phase 1: Monorepo scaffold" --description="Moon config, bun workspaces, tooling packages, package stubs, CI workflow. First phase — gets the build pipeline working." --type=task --priority=1
bd create --title="Phase 2: @bigtop/shadcn" --description="Initialize shadcn/ui with Tailwind v4. Add initial primitives: Button, Badge, Tooltip, DropdownMenu, Dialog, Input, ScrollArea. Map CSS variables to --bigtop-*." --type=task --priority=2
bd create --title="Phase 3: @bigtop/core" --description="Shell framework: ThemeProvider, Shell layout, CommandRegistry/Palette (cmdk), KeybindingManager (tinykeys), Notifications (sonner), Settings, PlatformApi." --type=task --priority=2
bd create --title="Phase 4: @bigtop/dockview" --description="Dockview integration: BigtopDockview wrapper, PanelRegistry, CustomTab, LayoutPersistence, theme.css overrides." --type=task --priority=2
bd create --title="Phase 5: sandbox/nenju" --description="Nenju web app consuming @bigtop/core + @bigtop/dockview. BeadsApi, Hono server, REST+WebSocket, core components." --type=task --priority=3
```

- [ ] **Step 3: Wire dependencies**

Each phase story depends on the previous. Phase 2-5 are children of the epic. Phase 1 is also a child.

```bash
bd dep add <phase2-id> <phase1-id>
bd dep add <phase3-id> <phase2-id>
bd dep add <phase4-id> <phase3-id>
bd dep add <phase5-id> <phase4-id>
```

- [ ] **Step 4: Claim Phase 1**

```bash
bd update <epic-id> --claim
bd update <phase1-id> --claim
```

---

### Task 2: @bigtop/typescript-config

**Files:**
- Create: `tooling/typescript-config/package.json`
- Create: `tooling/typescript-config/moon.yml`
- Create: `tooling/typescript-config/base.json`
- Create: `tooling/typescript-config/library.json`
- Create: `tooling/typescript-config/app.json`
- Create: `tooling/typescript-config/react-library.json`
- Create: `tooling/typescript-config/react-app.json`

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@bigtop/typescript-config",
  "version": "0.0.0",
  "private": true,
  "type": "module"
}
```

- [ ] **Step 2: Create moon.yml**

```yaml
language: 'typescript'
tags: ['tooling']
toolchain:
  default: 'bun'
```

- [ ] **Step 3: Create base.json**

Strict, ESM, bundler resolution, ES2024 target, verbatimModuleSyntax. No outDir/declaration — presets add those.

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "compilerOptions": {
    "target": "ES2024",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2024"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "verbatimModuleSyntax": true,
    "noUncheckedIndexedAccess": true
  },
  "exclude": ["node_modules", "dist"]
}
```

- [ ] **Step 4: Create library.json**

Extends base. Adds declaration emit. Does NOT set outDir — each package sets its own (tsconfig path resolution is relative to the defining file).

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "./base.json",
  "compilerOptions": {
    "declaration": true,
    "declarationMap": true
  }
}
```

- [ ] **Step 5: Create app.json**

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "./base.json",
  "compilerOptions": {
    "noEmit": true
  }
}
```

- [ ] **Step 6: Create react-library.json**

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "./library.json",
  "compilerOptions": {
    "jsx": "react-jsx"
  }
}
```

- [ ] **Step 7: Create react-app.json**

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "./app.json",
  "compilerOptions": {
    "jsx": "react-jsx"
  }
}
```

- [ ] **Step 8: Commit**

```bash
git add tooling/typescript-config/
git commit -m "feat: add @bigtop/typescript-config with base/library/app/react presets"
```

---

### Task 3: @bigtop/eslint-config

**Files:**
- Create: `tooling/eslint-config/package.json`
- Create: `tooling/eslint-config/moon.yml`
- Create: `tooling/eslint-config/base.js`
- Create: `tooling/eslint-config/react.js`

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@bigtop/eslint-config",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "exports": {
    "./base": "./base.js",
    "./react": "./react.js"
  },
  "dependencies": {
    "@eslint/js": "^10.0.0",
    "eslint-plugin-react-hooks": "^7.0.0",
    "typescript-eslint": "^8.57.0"
  },
  "peerDependencies": {
    "eslint": ">=9.0.0",
    "typescript": ">=5.0.0"
  }
}
```

- [ ] **Step 2: Create moon.yml**

```yaml
language: 'typescript'
tags: ['tooling']
toolchain:
  default: 'bun'
```

- [ ] **Step 3: Create base.js**

ESLint 10 flat config with typescript-eslint strict rules.

```js
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.strict,
  {
    rules: {
      '@typescript-eslint/consistent-type-imports': 'error',
    },
  },
);
```

- [ ] **Step 4: Create react.js**

Extends base, adds react-hooks plugin.

```js
import reactHooks from 'eslint-plugin-react-hooks';
import base from './base.js';

export default [
  ...base,
  {
    plugins: {
      'react-hooks': reactHooks,
    },
    rules: {
      'react-hooks/rules-of-hooks': 'error',
      'react-hooks/exhaustive-deps': 'warn',
    },
  },
];
```

- [ ] **Step 5: Commit**

```bash
git add tooling/eslint-config/
git commit -m "feat: add @bigtop/eslint-config with base and react flat configs"
```

---

### Task 4: Package Stubs (shadcn, core, dockview)

All three packages follow the same structure: package.json, moon.yml, tsconfig.json, eslint.config.js, src/index.ts. Library moon tasks from the design spec.

**Files (per package):**
- Create: `packages/<name>/package.json`
- Create: `packages/<name>/moon.yml`
- Create: `packages/<name>/tsconfig.json`
- Create: `packages/<name>/eslint.config.js`
- Create: `packages/<name>/src/index.ts`

- [ ] **Step 1: Create packages/shadcn/**

**package.json:**
```json
{
  "name": "@bigtop/shadcn",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    }
  },
  "devDependencies": {
    "@bigtop/eslint-config": "workspace:*",
    "@bigtop/typescript-config": "workspace:*",
    "eslint": "^10.0.0",
    "typescript": "^6.0.0"
  }
}
```

**moon.yml:**
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

**tsconfig.json:**
```json
{
  "extends": "@bigtop/typescript-config/library.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist"
  },
  "include": ["src"]
}
```

**eslint.config.js:**
```js
import base from '@bigtop/eslint-config/base';

export default [...base];
```

**src/index.ts:**
```ts
export {};
```

- [ ] **Step 2: Create packages/core/**

Same structure as shadcn. Only difference is package name.

**package.json:**
```json
{
  "name": "@bigtop/core",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    }
  },
  "devDependencies": {
    "@bigtop/eslint-config": "workspace:*",
    "@bigtop/typescript-config": "workspace:*",
    "eslint": "^10.0.0",
    "typescript": "^6.0.0"
  }
}
```

**moon.yml:** Same as shadcn.
**tsconfig.json:** Same as shadcn.
**eslint.config.js:** Same as shadcn.
**src/index.ts:** `export {};`

- [ ] **Step 3: Create packages/dockview/**

Same structure as shadcn/core. Only difference is package name.

**package.json:**
```json
{
  "name": "@bigtop/dockview",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    }
  },
  "devDependencies": {
    "@bigtop/eslint-config": "workspace:*",
    "@bigtop/typescript-config": "workspace:*",
    "eslint": "^10.0.0",
    "typescript": "^6.0.0"
  }
}
```

**moon.yml:** Same as shadcn.
**tsconfig.json:** Same as shadcn.
**eslint.config.js:** Same as shadcn.
**src/index.ts:** `export {};`

- [ ] **Step 4: Commit**

```bash
git add packages/
git commit -m "feat: add @bigtop/shadcn, @bigtop/core, @bigtop/dockview package stubs"
```

---

### Task 5: Root Config + CI

**Files:**
- Create: `tsconfig.json`
- Create: `bunfig.toml`
- Create: `.github/workflows/ci.yml`
- Modify: `.superset/setup.sh`

- [ ] **Step 1: Create root tsconfig.json**

Project references for IDE navigation. Not used for `tsc --build`.

```json
{
  "files": [],
  "references": [
    { "path": "packages/core" },
    { "path": "packages/dockview" },
    { "path": "packages/shadcn" }
  ]
}
```

- [ ] **Step 2: Create bunfig.toml**

```toml
[install]
peer = true
```

- [ ] **Step 3: Create .github/workflows/ci.yml**

```yaml
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

- [ ] **Step 4: Commit**

```bash
git add tsconfig.json bunfig.toml .github/
git commit -m "feat: add root tsconfig, bunfig.toml, and GitHub Actions CI"
```

---

### Task 6: Install + Verify

- [ ] **Step 1: Install dependencies**

Moon manages bun. Run any moon task to trigger dependency installation:

```bash
moon run :typecheck
```

- [ ] **Step 2: Fix any issues**

If typecheck or lint fail, diagnose and fix. Common issues:
- Missing `composite: true` (shouldn't be needed since we don't use `--build`)
- TypeScript 6 removing/renaming options
- ESLint 10 API changes

- [ ] **Step 3: Verify lint**

```bash
moon run :lint
```

- [ ] **Step 4: Verify setup.sh**

```bash
.superset/setup.sh
```

- [ ] **Step 5: Close Phase 1 bead**

```bash
bd close <phase1-id> --reason="Scaffold complete: tooling configs, package stubs, CI workflow. All typecheck+lint passing."
```
