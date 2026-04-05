# ESLint Suggestions and Immutable Data Structure Rules

**Date**: 2026-03-15
**Status**: Draft
**Scope**: Add ESLint suggestions (editor quick-fixes) to 3 existing rules, add suggestions to `prefer-list`, and create new `prefer-functype-map` and `prefer-functype-set` rules for functype immutable data structures

## Problem

The most frequently triggered rules in consumer codebases (`prefer-option`, `no-imperative-loops`, `prefer-either`) produce warnings but offer no automated remediation. Additionally, `functional/immutable-data` warnings (array `.push()`, Map `.set()`, object mutation) tell developers "don't mutate" but don't guide them toward functype's immutable alternatives (`List`, `Map`, `Set`). This creates friction during adoption.

## Decision: Suggestions over Autofix

All three rules will use ESLint **suggestions** (`hasSuggestions: true`) rather than autofix (`fixable: "code"`). Rationale:

- These transformations change runtime semantics, not just syntax
- Type annotation changes alone produce broken intermediate states (e.g., `Option<string>` type with `null` value)
- Suggestions appear as quick-fixes in the editor without auto-applying via `eslint --fix`
- Developers review each change individually, which is appropriate for functional migration

**Important**: Suggestions intentionally produce code that may have type errors until the developer completes the migration (e.g., changing a type to `Option<string>` while the value is still `null`). This is expected — the suggestion handles the mechanical part, the developer handles the semantic part.

## Rule 1: `prefer-option`

### Suggestion: Rewrite type annotation

Rewrites nullable union types to `Option<T>`. Only fires when exactly one non-null type exists in the union (`nonNullTypes.length === 1`). Unions like `string | number | null` are not candidates.

| Before | After |
|--------|-------|
| `string \| null` | `Option<string>` |
| `string \| undefined` | `Option<string>` |
| `string \| null \| undefined` | `Option<string>` |
| `{ name: string } \| null` | `Option<{ name: string }>` |
| `ReturnType<typeof setTimeout> \| undefined` | `Option<ReturnType<typeof setTimeout>>` |

### Suggestion: Add functype import

When `Option` is not already imported:
- If an existing `import { ... } from "functype"` exists, append `Option` to the specifiers
- Otherwise, add `import { Option } from "functype"` at the top of the file

### Out of scope (developer responsibility)

- Changing value assignments (`= null` to `= None`)
- Changing comparisons (`=== null` to `.isNone()`)
- Updating call sites that consume the changed type

### New messages

- `suggestOptionType`: "Replace with Option<{{type}}>"
- `suggestAddImport`: "Add {{symbol}} import from functype"

## Rule 2: `no-imperative-loops`

### Pattern-matched suggestions

Simple loop bodies (single statement or single push) get targeted suggestions. Complex bodies get the warning only. No functype import is needed — these suggestions use native JS methods (`.forEach`, `.map`, `.reduce`).

| Pattern | Suggestion |
|---------|-----------|
| `for (const x of arr) { sideEffect(x) }` | `arr.forEach((x) => { sideEffect(x) })` |
| `for (const x of arr) { results.push(transform(x)) }` | `arr.map((x) => transform(x))` |
| `for (const key in obj) { ... }` | `Object.keys(obj).forEach((key) => { ... })` |
| `for (let i = 0; i < n; i++)` | No suggestion (index-based access patterns are too varied) |
| `while` / `do..while` | No suggestion (too context-dependent) |

**Dropped from scope**: `for` loop with `+=` / accumulator → `.reduce()`. This requires identifying the accumulator variable declaration outside the loop, determining the initial value, and extracting the accumulation expression — too many failure modes for an initial implementation.

### Scope boundaries

- Only single-statement loop bodies get suggestions (one expression statement or one `.push()` call)
- Loops with `break`, `continue`, conditionals, or multiple statements: warning only
- Destructuring loop variables (`for (const { x, y } of items)`): warning only, no suggestion — the `.map()` / `.forEach()` parameter shape is ambiguous
- Nested loops: each loop gets its own suggestion independently
- The existing `getSuggestionForForLoop` function will be replaced with new AST-based analysis that returns fixer operations (the current implementation returns human-readable strings, which cannot drive code transformations)

### New messages

- `suggestForEach`: "Replace with {{iterable}}.forEach(...)"
- `suggestMap`: "Replace with {{iterable}}.map(...)"
- `suggestObjectKeys`: "Replace with Object.keys({{object}}).forEach(...)"

## Rule 3: `prefer-either`

### Suggestion types

| Pattern | Suggestion | Condition |
|---------|-----------|-----------|
| `try { expr } catch (e) { ... }` | `Try(() => expr)` | Single-expression try body AND simple catch (see criteria below) |
| `try { await expr } catch (e) { ... }` | `Try.fromPromise(expr)` | Same as above; `Try.fromPromise` accepts a Promise and returns `Try<T>` |
| `throw new Error(msg)` | `return Either.left(new Error(msg))` | `throw` is a direct statement in the enclosing function/method/arrow body, or a direct child of an `if` block in that body. Not offered when nested inside loops, nested conditionals, or other control structures where `return` would behave differently than `throw` |
| `throw nonErrorValue` | `return Either.left(new Error(String(nonErrorValue)))` | Same as above. Note: this wraps the value in `Error`, changing the error type — this is intentional as throwing non-Errors is bad practice |

### Catch block criteria for Try() suggestion

The `Try()` / `Try.fromPromise()` suggestion is only offered when the catch block meets one of these criteria:
- Empty catch block (`catch (e) {}`)
- Catch block contains only a `return` statement
- Catch block contains only a single expression statement (e.g., `console.error(e)`)

Catch blocks with multiple statements, conditional logic, cleanup operations, or re-assignments: warning only, no suggestion. The catch block's side effects would be silently dropped by `Try()`, which is unacceptable.

### Suggestion: Add functype import

When `Try` or `Either` is not already imported, adds the appropriate import following the same shared utility as `prefer-option`.

### New messages

- `suggestTry`: "Replace with Try(() => ...)"
- `suggestTryFromPromise`: "Replace with Try.fromPromise(...)"
- `suggestEitherLeft`: "Replace with Either.left(...)"
- `suggestAddImport`: "Add {{symbol}} import from functype"

## Rule 4: `prefer-list` (existing rule — add suggestions)

The `prefer-list` rule already flags `string[]`, `Array<T>`, `ReadonlyArray<T>` type annotations and array literals. Currently has no suggestions.

### Suggestion: Rewrite type annotation

| Before | After |
|--------|-------|
| `string[]` | `List<string>` |
| `Array<string>` | `List<string>` |
| `ReadonlyArray<string>` | `List<string>` |
| `{ name: string }[]` | `List<{ name: string }>` |

### Suggestion: Rewrite array literal

| Before | After |
|--------|-------|
| `["a", "b", "c"]` | `List.of("a", "b", "c")` |
| `[1, 2, 3]` | `List.of(1, 2, 3)` |

Arrays with spread elements (`[...existing, newItem]`) get no suggestion — `List.of(...existing, newItem)` changes semantics if `existing` is not iterable in the expected way.

### Suggestion: Add functype import

When `List` is not already imported, uses the shared `createImportFixer` utility.

### New messages

- `suggestListType`: "Replace with List<{{type}}>"
- `suggestListOf`: "Replace with List.of(...)"
- `suggestAddImport`: "Add {{symbol}} import from functype"

## Rule 5: `prefer-functype-map` (new rule)

New rule that flags native `Map` usage and suggests functype's immutable `Map`.

### Detection patterns

| Pattern | Message |
|---------|---------|
| `new Map()` / `new Map([...])` | Prefer `Map(...)` or `Map.of(...)` from functype |
| `Map<K, V>` type annotation (native) | Prefer functype `Map<K, V>` |

### Distinguishing native vs functype Map

The rule skips reporting when:
- `Map` is already imported from `"functype"` (checked via `hasFunctypeSymbol`)
- The `Map` usage is inside a functype context (checked via `isAlreadyUsingFunctype`)

### Suggestions

| Before | After | Condition |
|--------|-------|-----------|
| `new Map()` | `Map.empty()` | No arguments |
| `new Map([["a", 1], ["b", 2]])` | `Map.of(["a", 1], ["b", 2])` | Argument is an array literal of tuple literals |
| `new Map(someIterable)` | `Map(someIterable)` | Argument is a non-literal expression (variable, call, etc.) |
| Type annotation `Map<string, number>` | `Map<string, number>` (same syntax, different import) | Always |

### Suggestion: Add functype import

When `Map` is not already imported from functype, adds the import. Since `Map` shadows the global, the import is sufficient to switch semantics.

### New messages

- `preferFunctypeMap`: "Prefer functype Map<{{keyType}}, {{valueType}}> over native Map"
- `preferFunctypeMapLiteral`: "Prefer Map.of(...) or Map.empty() over new Map()"
- `suggestMapEmpty`: "Replace with Map.empty()"
- `suggestMapOf`: "Replace with Map.of(...)"
- `suggestMapFrom`: "Replace with Map(...)"
- `suggestAddImport`: "Add {{symbol}} import from functype"

### Rule options

```typescript
{
  type: "object",
  properties: {
    allowInTests: { type: "boolean", default: true }
  }
}
```

### Config placement

- `recommended`: "warn"
- `strict`: "error"

## Rule 6: `prefer-functype-set` (new rule)

New rule that flags native `Set` usage and suggests functype's immutable `Set`.

### Detection patterns

| Pattern | Message |
|---------|---------|
| `new Set()` / `new Set([...])` | Prefer `Set(...)` or `Set.of(...)` from functype |
| `Set<T>` type annotation (native) | Prefer functype `Set<T>` |

### Distinguishing native vs functype Set

Same approach as `prefer-map` — skip when `Set` is imported from `"functype"`.

### Suggestions

| Before | After | Condition |
|--------|-------|-----------|
| `new Set()` | `Set.empty()` | No arguments |
| `new Set(["a", "b", "c"])` | `Set.of("a", "b", "c")` | Argument is an array literal (elements extracted as variadic args) |
| `new Set([1, 2, 3])` | `Set.of(1, 2, 3)` | Same — array literal unwrapped |
| `new Set(someIterable)` | `Set(someIterable)` | Argument is a non-literal expression |

### Suggestion: Add functype import

Same shared utility as other rules.

### New messages

- `preferFunctypeSet`: "Prefer functype Set<{{type}}> over native Set"
- `preferFunctypeSetLiteral`: "Prefer Set.of(...) or Set.empty() over new Set()"
- `suggestSetEmpty`: "Replace with Set.empty()"
- `suggestSetOf`: "Replace with Set.of(...)"
- `suggestSetFrom`: "Replace with Set(...)"
- `suggestAddImport`: "Add {{symbol}} import from functype"

### Rule options

```typescript
{
  type: "object",
  properties: {
    allowInTests: { type: "boolean", default: true }
  }
}
```

### Config placement

- `recommended`: "warn"
- `strict`: "error"

## Shared Infrastructure: Import Fixer Utility

New file: `src/utils/import-fixer.ts`

Two exported functions:

### `createImportFixer(sourceCode, symbolName): (fixer: Rule.RuleFixer) => Rule.Fix`

Returns a **factory function** compatible with ESLint's `suggest[].fix` callback signature. When invoked with a fixer:
1. Checks if `symbolName` is already imported from `"functype"`
2. If a namespace import exists (`import * as F from "functype"`), returns `null` (no fix — adding a named import would conflict)
3. If an existing named import exists, appends `symbolName` to its specifiers
4. If no functype import exists, adds `import { symbolName } from "functype"` after the last import or at the top of the file

Usage in a rule:
```typescript
suggest: [{
  messageId: "suggestAddImport",
  data: { symbol: "Option" },
  fix: createImportFixer(sourceCode, "Option")
}]
```

### `hasFunctypeSymbol(sourceCode, symbolName): boolean`

Returns whether `symbolName` is already present in a functype import declaration.

This utility is shared across `prefer-option`, `prefer-either`, `prefer-list`, `prefer-map`, and `prefer-set`. The `no-imperative-loops` rule does not need it since its suggestions use native JS methods.

## Test Plan

Each rule gets new test cases using the ESLint rule tester `suggestions` format:

```typescript
{
  code: "const value: string | null = null",
  errors: [{
    messageId: "preferOption",
    suggestions: [{
      messageId: "suggestOptionType",
      // Note: output has a type error (Option<string> assigned null) — this is expected.
      // The suggestion handles the type annotation; the developer handles the value.
      output: "const value: Option<string> = null"
    }]
  }]
}
```

### Test coverage per rule

**prefer-option** (~6 new cases):
- Simple nullable: `string | null` -> suggestion offered
- Undefined: `string | undefined` -> suggestion offered
- Both: `string | null | undefined` -> suggestion offered
- Complex type: `{ name: string } | null` -> suggestion offered
- Multi-type union: `string | number | null` -> no suggestion (more than one non-null type)
- Already Option: no suggestion
- Import addition when missing

**no-imperative-loops** (~8 new cases):
- Simple for..of with single side effect -> forEach suggestion
- for..of with single push -> map suggestion
- for..in with single statement -> Object.keys suggestion
- Complex body (multiple statements) -> no suggestion, just warning
- Loop with break/continue -> no suggestion
- Destructuring loop variable -> no suggestion
- while/do..while -> no suggestion

**prefer-either** (~7 new cases):
- Simple try/catch with single expression, empty catch -> Try() suggestion
- Simple try/catch with single expression, catch with return -> Try() suggestion
- Async try/catch with single expression -> Try.fromPromise() suggestion
- try/catch with multi-statement catch -> no suggestion
- throw new Error in function body -> Either.left() suggestion
- throw new Error nested in loop -> no suggestion
- throw string literal -> Either.left(new Error()) suggestion
- Multi-statement try body -> no suggestion
- Import addition when missing

**prefer-list** (~5 new cases):
- `string[]` type -> List<string> suggestion
- `Array<string>` type -> List<string> suggestion
- Array literal -> List.of() suggestion
- Already using List: no suggestion
- Import addition when missing

**prefer-functype-map** (~8 new cases):
- `new Map()` -> Map.empty() suggestion
- `new Map([...])` with array literal -> Map.of() suggestion
- `new Map(someVar)` with non-literal arg -> Map(someVar) suggestion
- `Map<K, V>` type annotation -> suggestion with import
- Already imported from functype: no report
- Inside functype context: no report
- Import addition when missing
- Namespace import (`import * as`) -> no suggestion

**prefer-functype-set** (~7 new cases):
- `new Set()` -> Set.empty() suggestion
- `new Set([...])` with array literal -> Set.of() suggestion (elements unwrapped)
- `new Set(someVar)` with non-literal arg -> Set(someVar) suggestion
- `Set<T>` type annotation -> suggestion with import
- Already imported from functype: no report
- Import addition when missing
- Namespace import -> no suggestion

## Files Changed

| File | Change |
|------|--------|
| `src/utils/import-fixer.ts` | **New** — shared import fixer factory utility |
| `src/utils/functype-detection.ts` | Add `Map`, `Set` to known types list and static method patterns |
| `src/rules/prefer-option.ts` | Add `hasSuggestions`, `suggest` array with type rewrite + import |
| `src/rules/no-imperative-loops.ts` | Add `hasSuggestions`, `suggest` array with pattern-matched replacements. Replace `getSuggestionForForLoop` with AST-based analysis |
| `src/rules/prefer-either.ts` | Add `hasSuggestions`, `suggest` array with Try/Either.left replacements |
| `src/rules/prefer-list.ts` | Add `hasSuggestions`, `suggest` array with type + literal rewrite |
| `src/rules/prefer-functype-map.ts` | **New** — prefer functype Map over native Map |
| `src/rules/prefer-functype-set.ts` | **New** — prefer functype Set over native Set |
| `src/rules/index.ts` | Register `prefer-functype-map` and `prefer-functype-set` rules |
| `src/configs/recommended.ts` | Add `prefer-functype-map` and `prefer-functype-set` as "warn" |
| `src/configs/strict.ts` | Add `prefer-functype-map` and `prefer-functype-set` as "error" |
| `tests/rules/prefer-option.test.ts` | Add ~6 suggestion test cases |
| `tests/rules/no-imperative-loops.test.ts` | Add ~8 suggestion test cases |
| `tests/rules/prefer-either.test.ts` | Add ~7 suggestion test cases |
| `tests/rules/prefer-list.test.ts` | Add ~6 suggestion test cases (incl. ReadonlyArray, spread element edge case) |
| `tests/rules/prefer-functype-map.test.ts` | **New** — ~8 test cases (incl. non-literal arg) |
| `tests/rules/prefer-functype-set.test.ts` | **New** — ~7 test cases (incl. non-literal arg) |

## Implementation Order

1. `import-fixer.ts` utility (no dependencies)
2. Update `functype-detection.ts` to include `Map`, `Set`, `Stack` in known types
3. `prefer-option` suggestions (simplest — mechanical type rewrite)
4. `prefer-list` suggestions (similar to prefer-option — type + literal rewrites)
5. `prefer-functype-map` new rule + suggestions (native Map detection + functype Map suggestions)
6. `prefer-functype-set` new rule + suggestions (mirrors prefer-functype-map pattern)
7. `no-imperative-loops` suggestions (medium — pattern matching on loop bodies)
8. `prefer-either` suggestions (medium — Try/Either patterns with catch/throw constraints)
9. Register new rules in index, recommended, and strict configs
10. Tests for all rules
11. `pnpm validate` to verify everything passes
