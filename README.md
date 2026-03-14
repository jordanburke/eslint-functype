# eslint-functype

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Monorepo for ESLint tooling for functional TypeScript with [functype](https://github.com/jordanburke/functype).

## Packages

| Package | Version | Description |
| ------- | ------- | ----------- |
| [eslint-config-functype](./packages/config) | [![npm](https://img.shields.io/npm/v/eslint-config-functype.svg)](https://www.npmjs.com/package/eslint-config-functype) | Curated ESLint config bundle composing rules from eslint-plugin-functional, typescript-eslint, prettier, and import-sort |
| [eslint-plugin-functype](./packages/plugin) | [![npm](https://img.shields.io/npm/v/eslint-plugin-functype.svg)](https://www.npmjs.com/package/eslint-plugin-functype) | 9 custom ESLint rules for functype-specific patterns (prefer-option, prefer-either, prefer-fold, Do notation, etc.) |

## Quick Start

### Using both packages together (recommended)

```bash
pnpm add -D eslint-config-functype eslint-plugin-functype
```

```javascript
// eslint.config.mjs
import functypeConfig from "eslint-config-functype"
import functypePlugin from "eslint-plugin-functype"

export default [
  functypeConfig.configs.recommended,
  functypePlugin.configs.recommended,
  functypeConfig.configs.testOverrides, // relaxes FP rules in test files
]
```

### Using eslint-config-functype only (general FP rules)

```bash
pnpm add -D eslint-config-functype
```

```javascript
// eslint.config.mjs
import functypeConfig from "eslint-config-functype"

export default [
  functypeConfig.configs.recommended, // or .strict
]
```

### Using eslint-plugin-functype only (custom functype rules)

```bash
pnpm add -D eslint-plugin-functype
```

```javascript
// eslint.config.mjs
import functypePlugin from "eslint-plugin-functype"

export default [
  functypePlugin.configs.recommended, // or .strict
]
```

## eslint-config-functype

Curated configurations composing rules from upstream plugins. Two presets:

- **`recommended`** — balanced functional rules (warnings for mutations, `prefer-immutable-types` off, `no-throw-statements` with `allowToRejectWith`)
- **`strict`** — maximum enforcement (errors for mutations, explicit return types, strict boolean expressions)
- **`testOverrides`** — relaxes functional rules for test files (`*.test.ts`, `*.spec.ts`, test directories)

### Rules included

- **Core JS**: `prefer-const`, `no-var`, `no-throw-literal`, `object-shorthand`, `prefer-template`
- **TypeScript**: `consistent-type-imports`, `no-explicit-any`, `no-floating-promises`, `prefer-nullish-coalescing`, `prefer-optional-chain`
- **Functional** (eslint-plugin-functional): `no-let`, `immutable-data`, `no-throw-statements`, `no-try-statements`
- **Formatting**: prettier, simple-import-sort

## eslint-plugin-functype

9 custom ESLint rules for functype-specific patterns:

| Rule | Description | Auto-Fix |
| ---- | ----------- | -------- |
| `prefer-option` | Prefer `Option<T>` over `T \| null \| undefined` | Suggestion |
| `prefer-either` | Prefer `Either<E, T>` over try/catch | Suggestion |
| `prefer-list` | Prefer `List<T>` over native arrays | Suggestion |
| `prefer-fold` | Prefer `.fold()` over if/else chains | Yes |
| `prefer-map` | Prefer `.map()` over imperative transforms | Yes |
| `prefer-flatmap` | Prefer `.flatMap()` over `.map().flat()` | Yes |
| `no-get-unsafe` | Disallow unsafe `.get()` on Option/Either | No |
| `no-imperative-loops` | Prefer functional iteration | Suggestion |
| `prefer-do-notation` | Suggest Do notation for complex chains | Partial |

## Development

### Prerequisites

- Node.js >= 22.0.0
- pnpm 10.x

### Commands

```bash
pnpm install              # install all workspace dependencies
pnpm validate             # validate both packages (format, lint, typecheck, test, build)
pnpm validate:config      # validate config package only
pnpm validate:plugin      # validate plugin package only
```

### Per-package commands

```bash
cd packages/config && pnpm validate    # format + lint + typecheck + build
cd packages/plugin && pnpm validate    # format + lint + typecheck + test + build
cd packages/plugin && pnpm test        # run 116 tests
```

### Publishing

Each package is published independently:

```bash
cd packages/config && npm version patch && npm publish --access public
cd packages/plugin && npm version patch && npm publish --access public
```

## Related

- [functype](https://github.com/jordanburke/functype) — Functional programming library for TypeScript
- [ts-builds](https://github.com/jordanburke/ts-builds) — Standardized TypeScript build toolchain (provides ESLint config templates)

## License

[MIT](LICENSE)
