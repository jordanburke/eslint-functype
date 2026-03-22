# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Monorepo for ESLint tooling for functional TypeScript with the functype library. Contains two independently published npm packages:

- **`packages/config`** (`eslint-config-functype`) — Curated ESLint config composing rules from typescript-eslint, prettier, and import-sort. Exports `recommended`, `strict`, and `testOverrides` configs.
- **`packages/plugin`** (`eslint-plugin-functype`) — 9 custom ESLint rules for functype-specific patterns. Exports `recommended` and `strict` configs with self-referencing plugin binding.

## Development Commands

```bash
pnpm validate              # Root: validates both packages via ts-builds chains
pnpm validate:config       # Config only: format + lint + typecheck + build
pnpm validate:plugin       # Plugin only: format + lint + typecheck + test + build
```

### Per-package commands (run from package directory)

```bash
pnpm format / pnpm format:check    # Prettier
pnpm lint / pnpm lint:check        # ESLint
pnpm typecheck                     # TypeScript compilation check
pnpm test                          # Vitest (plugin only — config has no tests)
pnpm build                         # tsdown build to dist/
pnpm dev                           # Watch mode
```

### Running specific tests

```bash
cd packages/plugin && pnpm test -- tests/rules/prefer-option.test.ts
```

## Architecture

### Monorepo Structure

```
eslint-functype/
├── pnpm-workspace.yaml           # packages: ["packages/*"]
├── ts-builds.config.json          # workspace chains: validate → validate:config + validate:plugin
├── packages/
│   ├── config/                    # npm: eslint-config-functype
│   │   ├── src/configs/           # recommended.ts, strict.ts, test-overrides.ts
│   │   ├── src/cli/               # list-rules.ts CLI tool
│   │   ├── src/utils/             # dependency-validator.ts
│   │   └── tsdown.config.ts       # multi-entry: library + CLI (with banner)
│   └── plugin/                    # npm: eslint-plugin-functype
│       ├── src/rules/             # 9 custom ESLint rules
│       ├── src/configs/           # recommended.ts, strict.ts
│       ├── src/utils/             # functype-detection.ts
│       └── tests/                 # 138 tests across 12 suites
```

### Build System

- **ts-builds** — centralized toolchain for format, lint, typecheck, test, build
- **tsdown** — bundler (ESM output, sourcemaps, TypeScript declarations)
- **Vitest** — test runner (plugin package only)

### Config package (`packages/config`)

- Uses a custom `tsdown.config.ts` with two build targets: library entries + CLI with `#!/usr/bin/env node` banner
- Peer dependencies on typescript-eslint, prettier, import-sort (eslint-plugin-functional was removed in v2.1.0)
- `eslint.config.mjs` uses `ts-builds/eslint` base (NOT its own config) to avoid circular dependency

### Plugin package (`packages/plugin`)

- Rules in `src/rules/` each export a standard ESLint rule object
- `src/utils/functype-detection.ts` provides smart functype import/type detection to prevent false positives
- `src/index.ts` uses ESLint 9 self-referencing plugin pattern for `configs.recommended` and `configs.strict`
- Tests use `@typescript-eslint/rule-tester`

## Key Design Decisions

- **`eslint-plugin-functional` removed** (v2.1.0) — the functype plugin's own rules cover the same patterns with better specificity
- **`testOverrides` config** is an empty extension point for test files — consumers can layer their own test relaxations
- **Plugin configs** include self-referencing plugin binding — consumers use `functypePlugin.configs.recommended` without manual plugin registration
- **Packages are NOT merged** — different release cadences, different consumers (some use config without plugin)

## Adding New Rules to Plugin

1. Create rule file in `packages/plugin/src/rules/`
2. Add to `packages/plugin/src/rules/index.ts` exports
3. Add to configs (`recommended.ts` and/or `strict.ts`)
4. Create test file in `packages/plugin/tests/rules/`
5. Run `cd packages/plugin && pnpm validate`

## Publishing

Both packages publish via GitHub Actions with npm provenance on `v*` tags:

```bash
# Bump versions, commit, tag, push — CI publishes
cd packages/config && npm version patch --no-git-tag-version
cd packages/plugin && npm version patch --no-git-tag-version
git add -A && git commit -m "bump to vX.Y.Z"
git tag vX.Y.Z && git push --follow-tags
```

Or use the `/vbctp` skill which validates before bumping.
