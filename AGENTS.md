# Bigtop

## Conventions

- **No scripts in package.json** — moon handles all task execution
- **Moon manages bun** — don't run `bun install` manually
- **ESM-first** — `"type": "module"` everywhere

## Commands

```bash
moon run :build          # Build all packages
moon run :typecheck      # Type check all packages
moon run :lint           # Lint all packages
moon run :test           # Test all packages
moon run <project>:dev   # Dev server for a specific app
```
