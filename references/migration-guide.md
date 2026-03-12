# Migration Guide

Step-by-step process for safely migrating barrel imports to direct imports.

## Pre-Migration Checklist

Before starting any migration:

- [ ] Phase 1 analysis is complete — barrel files identified and classified
- [ ] Build baseline recorded (if Next.js) — First Load JS for all routes
- [ ] TypeScript compiles cleanly: `npx tsc --noEmit`
- [ ] Tests pass (if any): `npm test` / `npx jest` / `npx vitest`
- [ ] Git working tree is clean — commit or stash any pending changes
- [ ] Subpath exports verified — target import paths resolve correctly

## Import Resolution

### How Barrel Imports Work

```ts
// Consumer file
import { Button, useDebounce } from "@repo/ui";
```

Resolution chain:
1. Resolve `@repo/ui` → `packages/ui/src/index.ts` (barrel file)
2. Barrel re-exports `Button` from `./components/Button`
3. Barrel re-exports `useDebounce` from `./hooks/useDebounce`
4. **Problem:** Bundler must evaluate the ENTIRE barrel to find these exports
5. If tree-shaking fails, ALL 90+ modules behind the barrel are included

### How Direct Imports Work

```ts
// Consumer file
import { Button } from "@repo/ui/components/Button";
import { useDebounce } from "@repo/ui/hooks/useDebounce";
```

Resolution chain:
1. Resolve `@repo/ui/components/Button` → `packages/ui/src/components/Button.tsx`
2. Resolve `@repo/ui/hooks/useDebounce` → `packages/ui/src/hooks/useDebounce.ts`
3. **No barrel evaluated** — bundler only processes the exact files needed

## Transformation Rules

### Rule 1: Named Exports

**Before:**
```ts
import { Button, Card, Badge } from "@repo/ui";
```

**After:**
```ts
import { Button } from "@repo/ui/components/Button";
import { Card } from "@repo/ui/components/Card";
import { Badge } from "@repo/ui/components/Badge";
```

Each named export gets its own import statement pointing to the source module.

### Rule 2: Type Exports

**Before:**
```ts
import type { ButtonProps, CardProps } from "@repo/ui";
```

**After:**
```ts
import type { ButtonProps } from "@repo/ui/components/Button";
import type { CardProps } from "@repo/ui/components/Card";
```

Type imports have zero runtime cost (erased during compilation), so migration is optional but recommended for consistency.

### Rule 3: Mixed Imports (Values + Types)

**Before:**
```ts
import { Button, type ButtonProps, Card, type CardProps } from "@repo/ui";
```

**After:**
```ts
import { Button, type ButtonProps } from "@repo/ui/components/Button";
import { Card, type CardProps } from "@repo/ui/components/Card";
```

Group values and their associated types together by source module.

### Rule 4: Namespace Imports

**Before:**
```ts
import * as UI from "@repo/ui";
// Used as: UI.Button, UI.Card
```

**After:**
```ts
import { Button } from "@repo/ui/components/Button";
import { Card } from "@repo/ui/components/Card";
// Update usages: UI.Button → Button, UI.Card → Card
```

Namespace imports ALWAYS pull the entire barrel. Convert to named imports and update all usages in the file.

### Rule 5: Re-exports from Barrels

If a file re-exports from a barrel:

**Before:**
```ts
// src/lib/index.ts
export { Button, Card } from "@repo/ui";
```

**After:**
```ts
// src/lib/index.ts
export { Button } from "@repo/ui/components/Button";
export { Card } from "@repo/ui/components/Card";
```

This is the same transformation but applied to `export` statements instead of `import`.

## Subpath Exports Configuration

To enable direct imports, the library's `package.json` needs subpath exports:

```json
{
  "name": "@repo/ui",
  "exports": {
    ".": "./src/index.ts",
    "./components/*": "./src/components/*.tsx",
    "./icons/*": "./src/icons/*.tsx",
    "./hooks/*": "./src/hooks/*.ts",
    "./utils/*": "./src/utils/*.ts",
    "./constants/*": "./src/constants/*.ts"
  }
}
```

**Important considerations:**
- Keep the `"."` entry for backwards compatibility during migration
- Use wildcard patterns (`*`) to avoid listing every file
- Match the actual file extensions (`.tsx` for React components, `.ts` for plain TS)
- The consumer's `tsconfig.json` must use `"moduleResolution": "bundler"` or `"node16"` for subpath exports to work

## File-by-File Application Process

1. **Select next consumer file** from the mapping built in Phase 2
2. **Read the file** to understand current imports
3. **Build the transformation** — map each imported name to its direct path
4. **Show the diff to the user** — display BEFORE and AFTER
5. **Wait for confirmation** — do not apply without user approval
6. **Apply the change** — replace the import statement(s)
7. **Verify** — ensure TypeScript still compiles for this file
8. **Move to next file**

## Barrel File Cleanup

After all consumers have been migrated, decide what to do with the barrel file itself:

### Option A: Keep (Recommended during migration)

Leave the barrel file as-is. It still works, and any unmigrated consumers won't break. Remove it later once confident all consumers use direct imports.

### Option B: Remove

Delete the barrel file entirely. This is the cleanest approach but breaks any remaining consumers.

```bash
# Only after ALL consumers are confirmed migrated
rm src/components/index.ts
```

### Option C: Clean

Remove side effects from the barrel but keep the re-exports:

**Before:**
```ts
export * from "./Button";
export * from "./Card";
console.log("Components loaded"); // side effect
```

**After:**
```ts
export * from "./Button";
export * from "./Card";
```

## Edge Cases

### Circular Dependencies

Barrel files can mask circular dependencies. When module A imports from a barrel that re-exports module B, which imports from the same barrel for module A:

```
A → barrel → B → barrel → A  (circular!)
```

**Detection:** After migration, if TypeScript reports circular dependency errors or runtime shows `undefined` imports, a circular dependency was hidden by the barrel's lazy evaluation.

**Fix:** Identify the cycle and restructure — extract shared types/interfaces to a separate file that both modules can import from.

### Dynamic Imports

```ts
// This pulls the ENTIRE barrel at runtime
const module = await import("@repo/ui");
const Button = module.Button;
```

**Migration:**
```ts
// Direct dynamic import — only loads Button
const { Button } = await import("@repo/ui/components/Button");
```

### CSS and Asset Imports

```ts
// These are side-effect imports — do NOT remove
import "./styles.css";
import "@repo/ui/theme.css";
```

CSS imports have no named exports; they inject styles as a side effect. Always preserve these.

### Default Exports

```ts
// Before
import { default as ButtonComponent } from "@repo/ui";

// After
import ButtonComponent from "@repo/ui/components/Button";
```

### Monorepo Layered Packages

In monorepos with layered packages (e.g., `@repo/ui` depends on `@repo/primitives`):

```ts
// @repo/ui/components/Button.tsx
import { Box } from "@repo/primitives";  // Another barrel!
```

Migrate bottom-up: fix `@repo/primitives` barrels first, then `@repo/ui`.

### External Packages

**Do NOT rewrite imports for external packages** (lodash, date-fns, lucide-react, etc.). Instead:

```js
// next.config.js
module.exports = {
  experimental: {
    optimizePackageImports: ["lodash", "date-fns", "lucide-react"]
  }
}
```

Or use the package's own direct import paths if documented:
```ts
// lodash
import debounce from "lodash/debounce";  // NOT import { debounce } from "lodash"

// date-fns
import { format } from "date-fns/format";  // Direct subpath
```

## Post-Migration Verification

Run these checks after completing all migrations:

1. **TypeScript compilation:**
   ```bash
   npx tsc --noEmit
   ```

2. **Build:**
   ```bash
   npx next build    # Next.js
   npx vite build    # Vite
   npm run build     # Generic
   ```

3. **Tests:**
   ```bash
   npm test
   ```

4. **Bundle comparison:** Compare First Load JS before and after. Expected improvements:
   - Page chunks: 50-95% reduction (depending on how many unused modules were pulled in)
   - First Load JS: 20-40% reduction
   - If no improvement, check that barrels don't have side effects

5. **Runtime check:** Start the dev server and verify key pages render correctly.

## Rollback Plan

If migration causes issues:

1. **Git revert:** All changes should be in discrete commits
   ```bash
   git revert HEAD    # Revert last migration commit
   ```

2. **Restore barrel file:** If deleted
   ```bash
   git checkout HEAD~1 -- src/components/index.ts
   ```

3. **Verify build passes** after rollback

This is why we recommend:
- Committing after each file's migration
- Not deleting barrel files until all consumers are migrated
- Running builds after each batch of changes

## Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Deleting barrel before migrating all consumers | Breaks imports for unmigrated files | Keep barrel until all consumers migrated |
| Rewriting external package imports | Paths may not exist or may change across versions | Use `optimizePackageImports` instead |
| Ignoring side-effect imports | Removing `import "./styles.css"` breaks styling | Always preserve side-effect imports |
| Batch-applying without verification | One bad path breaks the entire build | Verify TypeScript compiles after each file |
| Forgetting to update re-exports | Intermediate barrels still pull everything | Transform `export { X } from` statements too |
