# barrel-exports

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that detects barrel export anti-patterns in TypeScript/React projects and safely migrates them to direct imports — reducing bundle size dramatically.

## The Problem

Barrel files (`index.ts` that re-export everything from a directory) look clean but force bundlers to load **all** modules even when you only use one. In a controlled Next.js monorepo test, a barrel import produced a **224 KB page chunk** vs **11.7 KB with direct imports** — a **19x difference**.

## What This Skill Does

### Phase 1: Analysis (read-only, always safe)

- Finds all `index.ts`/`index.tsx` barrel files in your project
- Classifies each one: pure barrel, mixed, with side effects
- Checks bundler and package configuration (`sideEffects`, `exports`, `optimizePackageImports`)
- Measures current bundle size (Next.js)
- Generates a risk-level report per file

### Phase 2: Migration (with confirmation)

- Rewrites barrel imports to direct paths (`from "@repo/ui"` → `from "@repo/ui/components/Button"`)
- Processes file by file, showing the diff before applying — never auto-applies
- Updates `package.json` and bundler config
- Runs builds before and after to show real impact with numbers

## Installation

```bash
npx skills add alexismunoz1/barrel-exports
```

## How to Use

After installing, just tell Claude what you need:

- **"My Next.js app bundle is too big"** — Runs analysis, finds barrels, measures impact
- **"Check if my project has barrel export problems"** — Analysis only, report without changes
- **"Migrate all my barrel imports to direct imports"** — Full analysis + file-by-file migration with confirmation

## License

MIT
