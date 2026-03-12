# Barrel Patterns & Anti-Patterns

Catalog of barrel export patterns, experiment data, and decision framework.

## Experiment Summary

### Setup

A controlled monorepo with two identical Next.js apps consuming the same `@repo/ui` package (90+ modules across components, icons, hooks, utils, and constants):

- **barrel-app** — imports via `import { Button, Card, ... } from "@repo/ui"` (barrel)
- **direct-app** — imports via `import { Button } from "@repo/ui/components/Button"` (direct)

Both apps use exactly the same 9 modules out of 90+ available. The only difference is the import style.

### Results

| Metric | Barrel App | Direct App | Difference |
|--------|-----------|------------|------------|
| First Load JS (/) | 155 kB | 106 kB | -31.6% |
| Page chunk (/) | 224 kB | 11.7 kB | **-94.8% (19x)** |
| Shared chunks | ~82 kB | ~94 kB | +14.6% |
| Modules evaluated | 90+ | 9 | -90% |

### Key Findings

1. **Page chunk is the critical metric** — 224 kB vs 11.7 kB. The barrel forced 90+ modules into the page chunk.
2. **Shared chunks increased slightly** for direct imports because framework code was no longer colocated with page code.
3. **Total First Load JS dropped 31.6%** — significant even though shared chunks grew.
4. **The barrel had a side effect** (`console.log`) that prevented tree-shaking entirely. Even without it, the star re-exports (`export *`) would have made tree-shaking unreliable.

### Build Output Evidence

**Barrel app route table:**
```
Route (app)                    Size     First Load JS
┌ ○ /                          224 kB       155 kB
└ ...
```

**Direct app route table:**
```
Route (app)                    Size     First Load JS
┌ ○ /                          11.7 kB      106 kB
└ ...
```

## 7 Barrel Anti-Patterns

### 1. The Root Barrel

```ts
// src/index.ts
export * from "./components";
export * from "./icons";
export * from "./hooks";
export * from "./utils";
export * from "./constants";
```

**Risk:** Critical
**Why:** Re-exports the entire library. Any import from this path evaluates all modules.
**Fix:** Add subpath exports to package.json. Consumers import from `@lib/components/Button` instead of `@lib`.

### 2. Nested Barrels (Barrel Chain)

```ts
// src/index.ts
export * from "./components";      // → components/index.ts (another barrel)

// src/components/index.ts
export * from "./Button";
export * from "./Card";
export * from "./Modal";
// ... 20 more
```

**Risk:** High
**Why:** Each level multiplies the evaluation cost. The bundler must resolve every barrel in the chain.
**Fix:** Flatten — consumers import directly from the leaf module.

### 3. Side-Effect Barrel

```ts
// src/index.ts
export * from "./components";
export * from "./hooks";

// Side effect: registration
console.log(`Library loaded: ${Date.now()}`);
(globalThis as any).__LIB_VERSION__ = "2.0.0";
```

**Risk:** Critical
**Why:** Side effects make the entire file un-tree-shakable. The bundler MUST include everything because it cannot prove the side effects are unused.
**Fix:** Remove side effects from the barrel. Move initialization to an explicit `init()` function.

### 4. Star Re-export Everything

```ts
// src/components/index.ts
export * from "./Button";
export * from "./Card";
export * from "./Modal";
export * from "./Table";
export * from "./Input";
// ... every component
```

**Risk:** High
**Why:** `export *` forces the bundler to evaluate every source module to discover what's exported. Named re-exports (`export { Button } from "./Button"`) are marginally better because the bundler knows the shape upfront, but tree-shaking is still unreliable for large barrels.
**Fix:** Use named re-exports at minimum. Prefer direct imports for consumers.

### 5. Mixed Barrel (Code + Re-exports)

```ts
// src/index.ts
export * from "./components";
export * from "./hooks";

// Original code mixed in
export const VERSION = "1.0.0";
export function init(config: Config) {
  // ...
}
```

**Risk:** Medium
**Why:** The original code (`VERSION`, `init`) makes the file both a barrel and a source module. Harder to reason about tree-shaking because the file has its own exports that may be referenced.
**Fix:** Extract original code to a separate module (e.g., `src/config.ts`). Keep the barrel as pure re-exports only, or eliminate it.

### 6. The "Convenience" Barrel

```ts
// src/components/forms/index.ts
export { Input } from "./Input";
export { Select } from "./Select";
export { Checkbox } from "./Checkbox";
// Only 3 exports — seemingly harmless
```

**Risk:** Low
**Why:** Small barrels with few exports are low-impact. Tree-shaking generally handles them well.
**When it becomes a problem:** When consumed by a parent barrel (`export * from "./forms"`), creating a chain.
**Fix:** Usually fine to keep. Address only if part of a chain or if the barrel has side effects.

### 7. The Default Export Barrel

```ts
// src/components/index.ts
export { default as Button } from "./Button";
export { default as Card } from "./Card";
export { default as Modal } from "./Modal";
```

**Risk:** Medium
**Why:** Re-exporting defaults as named exports is a common pattern but it requires the bundler to evaluate each source module to access the default export.
**Fix:** Consumers import defaults directly: `import Button from "@lib/components/Button"`.

## Next.js `optimizePackageImports`

### How It Works

When a package is listed in `optimizePackageImports`, Next.js:

1. Reads the package's barrel file to discover all exported names
2. Maps each name to its source file
3. At build time, rewrites `import { X } from "pkg"` → `import { X } from "pkg/source-file"`
4. The barrel file is never included in the bundle

### Limitations

- Must be explicitly listed (except for default-optimized packages)
- Does not handle side effects — if the barrel has side effects, they are silently dropped
- Only works with packages that have a discoverable module structure (flat or well-organized)
- Only applies at build time — dev mode may still evaluate barrels

### Default-Optimized Packages

Next.js automatically optimizes these packages without configuration:

```
@headlessui/react
@heroicons/react
@mui/icons-material
@mui/material
@tabler/icons-react
@tremor/react
@visx/*
antd
date-fns
firebase/auth, firebase/firestore (etc.)
lodash
lodash-es
lucide-react
ramda
react-bootstrap
recharts
rxjs
```

Check the [Next.js source](https://github.com/vercel/next.js/blob/canary/packages/next/src/lib/metadata/resolve-opengraph.ts) or docs for the current list.

### When to Use `optimizePackageImports` vs Direct Imports

| Scenario | Approach |
|----------|----------|
| External package (npm) | `optimizePackageImports` — don't modify node_modules |
| Internal monorepo package | Direct imports + subpath exports — you control the source |
| Package with side effects | Neither works well — remove side effects first |
| Package already optimized by default | No action needed |

## Impact by Module Type

Different module types have different barrel impact because of their size and import patterns:

| Module Type | Typical Size | Barrel Impact | Why |
|-------------|-------------|---------------|-----|
| Constants (data) | Large (10-100 KB) | **Critical** | Static data arrays (countries, timezones) are large and cannot be tree-shaken when pulled via barrel |
| Icons | Medium (1-5 KB each, many files) | **High** | Icon libraries often have 100+ icons; barrel pulls all of them |
| Components | Medium (5-20 KB each) | **High** | React components have dependencies (hooks, utils) that get pulled transitively |
| Utils | Small (1-5 KB each) | **Medium** | Individual utils are small but barrels pull all of them |
| Hooks | Small (1-10 KB each) | **Medium** | Similar to utils; usually few hooks but each may import other hooks |
| Types | Zero (erased) | **None** | TypeScript types are removed at compilation, never in bundle |

### Priority Order for Migration

1. **Constants** — highest ROI, largest files, zero tree-shaking benefit from barrel
2. **Icons** — many files, each small, but sum is large
3. **Components** — moderate size, transitive dependencies amplify impact
4. **Utils/Hooks** — lower individual impact but easy wins
5. **Types** — no runtime impact, migrate for consistency only

## Decision Matrix

Use this matrix to decide how to handle each barrel:

| Factor | Keep Barrel | Migrate to Direct | Use optimizePackageImports |
|--------|------------|-------------------|--------------------------|
| **Package type** | N/A | Internal / monorepo | External / npm |
| **Side effects** | Only if intentional | Remove first, then migrate | Won't help (side effects dropped) |
| **Export count** | <5 exports | >5 exports | Any count |
| **Consumer count** | 1-2 consumers | 3+ consumers | Many consumers |
| **Module sizes** | Small, uniform | Large or varied | Any |
| **Tree-shaking works** | Yes, verified | No, or unreliable | Partial |
| **Team size** | Solo / small | Any | Any |

### Quick Decision Flow

```
Is it an external npm package?
  YES → Add to optimizePackageImports (if Next.js) or use package's direct imports
  NO ↓

Does the barrel have side effects?
  YES → Remove side effects first, then re-evaluate
  NO ↓

Does the barrel re-export >20 modules?
  YES → Migrate to direct imports (high impact)
  NO ↓

Is the barrel part of a chain (nested barrels)?
  YES → Migrate to direct imports (chain amplifies impact)
  NO → Barrel is probably fine if it has no side effects; verify with build metrics
```
