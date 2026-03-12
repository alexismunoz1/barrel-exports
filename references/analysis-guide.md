# Analysis Guide

Detailed methodology for detecting and classifying barrel export patterns.

## What Is a Barrel File?

A barrel file is an `index.ts` (or `index.tsx`, `index.js`) that re-exports from other modules to provide a single import path:

```ts
// src/components/index.ts — a barrel file
export { Button } from "./Button";
export { Card } from "./Card";
export { Modal } from "./Modal";
export * from "./Badge";
```

**Why they exist:** Developer convenience — `import { Button, Card } from "@lib/components"` instead of individual imports.

**Why they hurt:** Bundlers must evaluate the entire barrel to resolve named exports. If tree-shaking fails (side effects, dynamic patterns, missing config), the entire module graph behind the barrel gets included.

### Canonical Barrel Patterns

```ts
// Pattern 1: Named re-exports
export { Button } from "./Button";
export { Card } from "./Card";

// Pattern 2: Star re-exports
export * from "./Button";
export * from "./Card";

// Pattern 3: Nested barrels (barrel importing from barrels)
export * from "./components";   // → components/index.ts → another barrel
export * from "./hooks";        // → hooks/index.ts → another barrel

// Pattern 4: Aggregation with rename
export { default as Button } from "./Button";
export { ButtonProps } from "./Button";
```

## Side Effect Detection

Side effects prevent tree-shaking. The bundler must keep all code from a module if it may have side effects.

### Side Effect Taxonomy

**Category 1: Global Mutations**
```ts
// Risk: CRITICAL — modifies global state on import
(globalThis as any).__REGISTRY__ = new Map();
window.APP_VERSION = "1.0.0";
Object.defineProperty(globalThis, "config", { value: {} });
```

**Category 2: Data Structure Initialization**
```ts
// Risk: HIGH — creates shared mutable state
const componentRegistry = new Map();
componentRegistry.set("Button", ButtonComponent);
export { componentRegistry };
```

**Category 3: Console Output**
```ts
// Risk: MEDIUM — side effect but usually harmless
console.log("Module loaded");
console.warn("Deprecated: use v2 API");
```

**Category 4: Function Calls at Module Level**
```ts
// Risk: HIGH — unknown side effects
registerComponents();
initializeTheme();
polyfill();
```

**Category 5: Class Instantiation**
```ts
// Risk: HIGH — constructor may have side effects
const logger = new Logger("ui");
const cache = new LRUCache({ max: 100 });
```

**Category 6: Event Listeners / Subscriptions**
```ts
// Risk: CRITICAL — modifies browser/runtime state
window.addEventListener("error", handleError);
document.addEventListener("DOMContentLoaded", init);
process.on("uncaughtException", handler);
```

### Classification Algorithm

For each `index.ts` / `index.tsx` file:

1. **Read the file contents**
2. **Classify each top-level statement:**
   - `export * from "X"` → re-export
   - `export { A, B } from "X"` → re-export
   - `export { default as X } from "X"` → re-export
   - `export type { X } from "X"` → type re-export (zero runtime cost)
   - `import "X"` → side-effect import
   - Any other statement → original code or side effect
3. **Classify the file:**
   - All statements are re-exports → **Pure barrel**
   - Mix of re-exports and original code → **Mixed barrel**
   - Re-exports + side effects detected → **Side-effect barrel**
   - No re-exports → **Entry point** (not a barrel)
4. **Score severity:**
   - Count total re-exports (more = higher impact)
   - Check if barrel imports from other barrels (nested = multiplied impact)
   - Check for side effects (any = tree-shaking blocked)

## Package.json Checks

### `exports` Field (Subpath Exports)

```json
// GOOD: Enables direct imports via subpaths
{
  "exports": {
    ".": "./src/index.ts",
    "./components/*": "./src/components/*.tsx",
    "./hooks/*": "./src/hooks/*.ts",
    "./utils/*": "./src/utils/*.ts"
  }
}

// BAD: Only root export — forces barrel usage
{
  "exports": {
    ".": "./src/index.ts"
  }
}

// BAD: No exports field — relies on main/module fields
{
  "main": "./src/index.ts"
}
```

## next.config Checks

### `optimizePackageImports`

```js
// next.config.js
module.exports = {
  experimental: {
    optimizePackageImports: ["@repo/ui", "lucide-react", "date-fns"]
  }
}
```

**What it does:** Next.js transforms barrel imports into direct imports at build time. The bundler rewrites `import { Button } from "@repo/ui"` into the equivalent of `import { Button } from "@repo/ui/components/Button"`.

**Limitations:**
- Only works for packages that have a discoverable module structure
- Does not handle side effects — if the barrel has side effects, they may be dropped
- Only applies to the packages listed — must be explicit
- Some packages are optimized by default (see `references/barrel-patterns.md`)

### `transpilePackages`

```js
// Required for monorepo internal packages
module.exports = {
  transpilePackages: ["@repo/ui", "@repo/utils"]
}
```

**What it does:** Tells Next.js to compile these packages through its bundler pipeline instead of treating them as pre-built externals. Without this, internal monorepo packages may not be tree-shaken at all.

## Build Output Parsing

### Next.js Route Table Format

After `next build`, Next.js prints a route table:

```
Route (app)                    Size     First Load JS
┌ ○ /                          5.2 kB       87.3 kB
├ ○ /about                     1.1 kB       83.2 kB
└ ○ /dashboard                 224 kB       306 kB
+ First Load JS shared by all  82.1 kB
  ├ chunks/framework-...        45.2 kB
  ├ chunks/main-...             28.4 kB
  └ chunks/pages/_app-...       8.5 kB
```

**Key columns:**
- **Size** — JS unique to that route (page chunk)
- **First Load JS** — Size + shared chunks = total JS for initial page load

**What to look for:**
- Page chunks >50 kB suggest barrel pollution (unused code pulled in)
- Compare similar pages — if one is 10x larger, barrels are likely the cause
- Shared chunks should be similar across barrel and direct import versions

## Reporting Format

Use this template for the analysis report:

```markdown
## Barrel Export Analysis Report

**Project:** {project name}
**Framework:** {Next.js X.X / React / Vite / etc.}
**Date:** {date}

### Summary

- **Barrel files found:** {N}
- **Total re-exports across all barrels:** {N}
- **Files with side effects:** {N}
- **Configuration issues:** {N}

### Barrel Files

| # | File | Type | Re-exports | Side Effects | Consumers | Risk |
|---|------|------|------------|-------------|-----------|------|
| 1 | src/index.ts | Pure barrel | 45 | None | 12 | High |
| 2 | src/components/index.ts | Side-effect | 20 | console.log | 8 | Critical |

### Configuration Status

| Check | Status | Details |
|-------|--------|---------|
| Subpath exports | ⚠️ Missing | Add `exports` field for direct import paths |
| optimizePackageImports | ✅ Present | @repo/ui is listed |
| transpilePackages | ✅ Present | @repo/ui is listed |
| moduleResolution | ✅ bundler | Supports subpath exports |

### Bundle Metrics (if measured)

| Route | Page Chunk | First Load JS |
|-------|-----------|---------------|
| / | 224 kB | 306 kB |
| /about | 1.1 kB | 83.2 kB |

### Recommendations

1. **[Priority]** {Specific action with expected impact}
2. ...
```
