# ESLint Suggestions & Immutable DS Rules — Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add ESLint editor suggestions to 4 existing rules and create 2 new rules for functype immutable data structures.

**Architecture:** Each rule's `create()` method gains a `suggest` array in its `context.report()` calls. A shared `import-fixer.ts` utility provides reusable import manipulation for all rules that need to add functype imports. Two new rules (`prefer-functype-map`, `prefer-functype-set`) follow the exact same structure as existing rules like `prefer-list`.

**Tech Stack:** ESLint Rule API (suggestions), @typescript-eslint/rule-tester, Vitest, tsdown

**Spec:** `docs/superpowers/specs/2026-03-15-eslint-suggestions-design.md`

---

## Chunk 1: Shared Infrastructure

### Task 1: Create import-fixer utility

**Files:**
- Create: `packages/plugin/src/utils/import-fixer.ts`
- Test: `packages/plugin/tests/utils/import-fixer.test.ts`

- [ ] **Step 1: Write failing tests for import-fixer**

```typescript
// packages/plugin/tests/utils/import-fixer.test.ts
import { describe, it, expect } from "vitest"
import { hasFunctypeSymbol } from "../../src/utils/import-fixer"

describe("import-fixer", () => {
  describe("hasFunctypeSymbol", () => {
    it("returns true when symbol is imported from functype", () => {
      // Will test via rule-tester integration since hasFunctypeSymbol needs sourceCode
    })
  })
})
```

Note: `createImportFixer` and `hasFunctypeSymbol` are best tested through the rule tests themselves (they require ESLint's `sourceCode` object). Create the utility file with the implementation directly — the rule tests in subsequent tasks serve as integration tests.

- [ ] **Step 2: Implement import-fixer.ts**

```typescript
// packages/plugin/src/utils/import-fixer.ts
import type { Rule, SourceCode } from "eslint"

/**
 * Check if a symbol is already imported from "functype"
 */
export function hasFunctypeSymbol(sourceCode: SourceCode, symbolName: string): boolean {
  const ast = sourceCode.ast
  for (const node of ast.body) {
    if (
      node.type === "ImportDeclaration" &&
      node.source.type === "Literal" &&
      node.source.value === "functype"
    ) {
      if (!node.specifiers) continue
      for (const spec of node.specifiers) {
        if (spec.type === "ImportSpecifier" && spec.imported.type === "Identifier" && spec.imported.name === symbolName) {
          return true
        }
        // Namespace import covers everything
        if (spec.type === "ImportNamespaceSpecifier") return true
      }
    }
  }
  return false
}

/**
 * Returns a fixer factory function compatible with ESLint suggest[].fix.
 * Adds `symbolName` to the functype import, or creates a new import.
 */
export function createImportFixer(
  sourceCode: SourceCode,
  symbolName: string,
): (fixer: Rule.RuleFixer) => Rule.Fix | null {
  return (fixer: Rule.RuleFixer): Rule.Fix | null => {
    if (hasFunctypeSymbol(sourceCode, symbolName)) return null

    const ast = sourceCode.ast

    // Find existing functype import
    for (const node of ast.body) {
      if (
        node.type === "ImportDeclaration" &&
        node.source.type === "Literal" &&
        node.source.value === "functype"
      ) {
        // Check for namespace import — can't add named imports alongside it
        const hasNamespace = node.specifiers?.some(
          (s: { type: string }) => s.type === "ImportNamespaceSpecifier",
        )
        if (hasNamespace) return null

        // Append to existing named import
        const lastSpecifier = node.specifiers?.[node.specifiers.length - 1]
        if (lastSpecifier) {
          return fixer.insertTextAfter(lastSpecifier, `, ${symbolName}`)
        }
      }
    }

    // No functype import exists — add new import after last import or at top
    const lastImport = [...ast.body].reverse().find((n) => n.type === "ImportDeclaration")
    if (lastImport) {
      return fixer.insertTextAfter(lastImport, `\nimport { ${symbolName} } from "functype"`)
    }

    // No imports at all — add at top of file
    return fixer.insertTextBefore(ast.body[0], `import { ${symbolName} } from "functype"\n\n`)
  }
}
```

- [ ] **Step 3: Commit**

```bash
cd packages/plugin
git add src/utils/import-fixer.ts
git commit -m "feat(plugin): add shared import-fixer utility for ESLint suggestions"
```

### Task 2: Update functype-detection.ts

**Files:**
- Modify: `packages/plugin/src/utils/functype-detection.ts:56` and `:99` and `:123-128`

- [ ] **Step 1: Add Map, Set to known types in getFunctypeImports**

In `getFunctypeImports`, update the types array at line 56:

```typescript
// Before:
if (["Option", "Either", "List", "LazyList", "Task", "Try"].includes(name)) {

// After:
if (["Option", "Either", "List", "LazyList", "Task", "Try", "Map", "Set", "Stack"].includes(name)) {
```

- [ ] **Step 2: Add Map, Set to isFunctypeType**

In `isFunctypeType`, update the fallback check at line 99:

```typescript
// Before:
return types.has(typeName) || ["Option", "Either", "List", "LazyList", "Task", "Try"].includes(typeName)

// After:
return types.has(typeName) || ["Option", "Either", "List", "LazyList", "Task", "Try", "Map", "Set", "Stack"].includes(typeName)
```

- [ ] **Step 3: Add Map, Set static methods to isFunctypeCall**

In `isFunctypeCall`, extend the static method checks after line 128:

```typescript
// Add after the List check:
(objectName === "Map" && ["of", "empty"].includes(methodName)) ||
(objectName === "Set" && ["of", "empty"].includes(methodName))
```

- [ ] **Step 4: Run existing tests to verify no regressions**

Run: `cd packages/plugin && pnpm test`
Expected: All 116 existing tests pass.

- [ ] **Step 5: Commit**

```bash
git add packages/plugin/src/utils/functype-detection.ts
git commit -m "feat(plugin): add Map, Set, Stack to functype detection utilities"
```

---

## Chunk 2: prefer-option & prefer-list suggestions

### Task 3: Add suggestions to prefer-option

**Files:**
- Modify: `packages/plugin/src/rules/prefer-option.ts`
- Modify: `packages/plugin/tests/rules/prefer-option.test.ts`

- [ ] **Step 1: Write failing suggestion tests**

Add to the `invalid` array in `packages/plugin/tests/rules/prefer-option.test.ts`:

```typescript
// After existing invalid cases, add suggestion test cases:
{
  name: "String or null should suggest Option<string>",
  code: "const value: string | null = null",
  errors: [{
    messageId: "preferOption",
    suggestions: [
      {
        messageId: "suggestOptionType",
        output: "const value: Option<string> = null",
      },
    ],
  }],
},
{
  name: "String or undefined should suggest Option<string>",
  code: "const value: string | undefined = undefined",
  errors: [{
    messageId: "preferOption",
    suggestions: [
      {
        messageId: "suggestOptionType",
        output: "const value: Option<string> = undefined",
      },
    ],
  }],
},
{
  name: "String with both null and undefined should suggest Option<string>",
  code: "const value: string | null | undefined = null",
  errors: [{
    messageId: "preferOption",
    suggestions: [
      {
        messageId: "suggestOptionType",
        output: "const value: Option<string> = null",
      },
    ],
  }],
},
{
  name: "Complex type should suggest Option",
  code: "const user: { name: string; age: number } | null = null",
  errors: [{
    messageId: "preferOption",
    suggestions: [
      {
        messageId: "suggestOptionType",
        output: "const user: Option<{ name: string; age: number }> = null",
      },
    ],
  }],
},
{
  name: "Should add import when Option not imported",
  code: "const value: string | null = null",
  errors: [{
    messageId: "preferOption",
    suggestions: [
      {
        messageId: "suggestOptionType",
        output: "const value: Option<string> = null",
      },
    ],
  }],
},
```

Also add to the `valid` array to verify multi-type unions are NOT flagged (the rule already handles this, but confirm no suggestions leak through):

```typescript
{
  name: "Multi-type union with null is not flagged",
  code: "const value: string | number | null = null",
},
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd packages/plugin && pnpm vitest run tests/rules/prefer-option.test.ts`
Expected: FAIL — `suggestions` not returned by rule.

- [ ] **Step 3: Implement suggestions in prefer-option**

Update `packages/plugin/src/rules/prefer-option.ts`:

Add `hasSuggestions: true` to meta. Add `suggestOptionType` to messages. Add `suggest` array to `context.report()`.

```typescript
import type { Rule } from "eslint"

import type { ASTNode } from "../types/ast"
import { getFunctypeImportsLegacy, isAlreadyUsingFunctype, isFunctypeType } from "../utils/functype-detection"
import { createImportFixer, hasFunctypeSymbol } from "../utils/import-fixer"

const rule: Rule.RuleModule = {
  meta: {
    type: "suggestion",
    hasSuggestions: true,
    docs: {
      description: "Prefer Option<T> over nullable types (T | null | undefined)",
      recommended: true,
    },
    schema: [
      {
        type: "object",
        properties: {
          allowNullableIntersections: {
            type: "boolean",
            default: false,
          },
        },
        additionalProperties: false,
      },
    ],
    messages: {
      preferOption: "Prefer Option<{{type}}> over nullable type '{{nullable}}'",
      preferOptionReturn: "Prefer Option<{{type}}> as return type over nullable '{{nullable}}'",
      suggestOptionType: "Replace with Option<{{type}}>",
      suggestAddImport: "Add {{symbol}} import from functype",
    },
  },

  create(context) {
    const functypeImports = getFunctypeImportsLegacy(context)
    const sourceCode = context.sourceCode

    return {
      TSUnionType(node: ASTNode) {
        if (!node.types || node.types.length < 2) return

        const hasNull = node.types.some(
          (type: ASTNode) => type.type === "TSNullKeyword" || type.type === "TSUndefinedKeyword",
        )

        if (!hasNull) return

        const nonNullTypes = node.types.filter(
          (type: ASTNode) => type.type !== "TSNullKeyword" && type.type !== "TSUndefinedKeyword",
        )

        if (nonNullTypes.length === 1) {
          const nonNullType = nonNullTypes[0]

          if (isFunctypeType(nonNullType, functypeImports)) return
          if (isAlreadyUsingFunctype(node, functypeImports)) return

          const nonNullTypeText = sourceCode.getText(nonNullType)
          const fullType = sourceCode.getText(node)

          if (nonNullTypeText.startsWith("Option<")) return

          const suggest: Rule.SuggestionReportDescriptor[] = [
            {
              messageId: "suggestOptionType",
              data: { type: nonNullTypeText },
              fix(fixer) {
                return fixer.replaceText(node, `Option<${nonNullTypeText}>`)
              },
            },
          ]

          // Add import suggestion if Option is not already imported
          if (!hasFunctypeSymbol(sourceCode, "Option")) {
            suggest.push({
              messageId: "suggestAddImport",
              data: { symbol: "Option" },
              fix: createImportFixer(sourceCode, "Option"),
            })
          }

          context.report({
            node,
            messageId: "preferOption",
            data: {
              type: nonNullTypeText,
              nullable: fullType,
            },
            suggest,
          })
        }
      },
    }
  },
}

export default rule
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd packages/plugin && pnpm vitest run tests/rules/prefer-option.test.ts`
Expected: All tests PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/plugin/src/rules/prefer-option.ts packages/plugin/tests/rules/prefer-option.test.ts
git commit -m "feat(plugin): add suggestions to prefer-option rule"
```

### Task 4: Add suggestions to prefer-list

**Files:**
- Modify: `packages/plugin/src/rules/prefer-list.ts`
- Modify: `packages/plugin/tests/rules/prefer-list.test.ts`

- [ ] **Step 1: Write failing suggestion tests**

Add to the `invalid` array in `packages/plugin/tests/rules/prefer-list.test.ts`:

```typescript
// Suggestion test cases
{
  name: "string[] should suggest List<string>",
  code: 'const items: string[] = ["a", "b", "c"]',
  errors: [{
    messageId: "preferList",
    suggestions: [
      {
        messageId: "suggestListType",
        output: 'const items: List<string> = ["a", "b", "c"]',
      },
    ],
  }],
},
{
  name: "Array<string> should suggest List<string>",
  code: 'const items: Array<string> = ["a", "b", "c"]',
  errors: [{
    messageId: "preferList",
    suggestions: [
      {
        messageId: "suggestListType",
        output: 'const items: List<string> = ["a", "b", "c"]',
      },
    ],
  }],
},
{
  name: "ReadonlyArray<string> should suggest List<string>",
  code: 'const items: ReadonlyArray<string> = ["a", "b", "c"]',
  errors: [{
    messageId: "preferList",
    suggestions: [
      {
        messageId: "suggestListType",
        output: 'const items: List<string> = ["a", "b", "c"]',
      },
    ],
  }],
},
{
  name: "Array literal should suggest List.of()",
  code: 'const items = ["a", "b", "c"]',
  errors: [{
    messageId: "preferListLiteral",
    suggestions: [
      {
        messageId: "suggestListOf",
        output: 'const items = List.of("a", "b", "c")',
      },
    ],
  }],
},
```

Also add to the `valid` array to verify spread elements are NOT given suggestions:

```typescript
// Spread elements should not crash or produce bad suggestions
// (The rule still flags the type, but the ArrayExpression handler skips spreads)
```

And add a test verifying array with spread gets no suggestion in `invalid`:

```typescript
{
  name: "Array literal with spread should have no suggestion",
  code: "const items = [...existing, newItem]",
  errors: [{
    messageId: "preferListLiteral",
    // No suggestions — spread elements make List.of() ambiguous
  }],
},
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd packages/plugin && pnpm vitest run tests/rules/prefer-list.test.ts`
Expected: FAIL — `suggestions` not returned.

- [ ] **Step 3: Implement suggestions in prefer-list**

Update `packages/plugin/src/rules/prefer-list.ts`:

1. Add `hasSuggestions: true` to meta
2. Add `suggestListType`, `suggestListOf`, `suggestAddImport` messages
3. Import `createImportFixer`, `hasFunctypeSymbol` from import-fixer
4. Add `suggest` arrays to each `context.report()` call

Key changes to each handler:
- `TSArrayType`: suggest replacing `string[]` with `List<string>` — `fixer.replaceText(node, \`List<${elementType}>\`)`
- `TSTypeReference` for `Array<T>` and `ReadonlyArray<T>`: suggest replacing with `List<T>`
- `ArrayExpression`: suggest replacing `[a, b, c]` with `List.of(a, b, c)` — skip if any element is a `SpreadElement`

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd packages/plugin && pnpm vitest run tests/rules/prefer-list.test.ts`
Expected: All tests PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/plugin/src/rules/prefer-list.ts packages/plugin/tests/rules/prefer-list.test.ts
git commit -m "feat(plugin): add suggestions to prefer-list rule"
```

---

## Chunk 3: New rules (prefer-functype-map, prefer-functype-set)

### Task 5: Create prefer-functype-map rule

**Files:**
- Create: `packages/plugin/src/rules/prefer-functype-map.ts`
- Create: `packages/plugin/tests/rules/prefer-functype-map.test.ts`

- [ ] **Step 1: Write tests for prefer-functype-map**

```typescript
// packages/plugin/tests/rules/prefer-functype-map.test.ts
import { describe } from "vitest"
import { ruleTester } from "../utils/rule-tester"
import rule from "../../src/rules/prefer-functype-map"

describe("prefer-functype-map", () => {
  ruleTester.run("prefer-functype-map", rule, {
    valid: [
      {
        name: "Functype Map import is allowed",
        code: 'import { Map } from "functype"\nconst m = Map.empty()',
      },
      {
        name: "Non-Map code is allowed",
        code: "const x: string = 'hello'",
      },
    ],
    invalid: [
      {
        name: "new Map() should suggest Map.empty()",
        code: "const m = new Map()",
        errors: [{
          messageId: "preferFunctypeMapLiteral",
          suggestions: [
            {
              messageId: "suggestMapEmpty",
              output: "const m = Map.empty()",
            },
          ],
        }],
      },
      {
        name: "new Map with array literal should suggest Map.of()",
        code: 'const m = new Map([["a", 1], ["b", 2]])',
        errors: [{
          messageId: "preferFunctypeMapLiteral",
          suggestions: [
            {
              messageId: "suggestMapOf",
              output: 'const m = Map.of(["a", 1], ["b", 2])',
            },
          ],
        }],
      },
      {
        name: "new Map with variable should suggest Map()",
        code: "const m = new Map(entries)",
        errors: [{
          messageId: "preferFunctypeMapLiteral",
          suggestions: [
            {
              messageId: "suggestMapFrom",
              output: "const m = Map(entries)",
            },
          ],
        }],
      },
      {
        name: "Map type annotation should be flagged",
        code: "const m: Map<string, number> = new Map()",
        errors: [
          { messageId: "preferFunctypeMap" },
          { messageId: "preferFunctypeMapLiteral" },
        ],
      },
    ],
  })
})
```

Also add to `valid` to cover namespace import edge case:

```typescript
{
  name: "Namespace import from functype should not be flagged",
  code: 'import * as F from "functype"\nconst m = new Map()',
  // Map is considered available via namespace — skip reporting
},
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd packages/plugin && pnpm vitest run tests/rules/prefer-functype-map.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement prefer-functype-map rule**

```typescript
// packages/plugin/src/rules/prefer-functype-map.ts
import type { Rule } from "eslint"

import type { ASTNode } from "../types/ast"
import { isAlreadyUsingFunctype, getFunctypeImportsLegacy } from "../utils/functype-detection"
import { createImportFixer, hasFunctypeSymbol } from "../utils/import-fixer"

const rule: Rule.RuleModule = {
  meta: {
    type: "suggestion",
    hasSuggestions: true,
    docs: {
      description: "Prefer functype Map over native Map for immutable key-value stores",
      recommended: true,
    },
    schema: [
      {
        type: "object",
        properties: {
          allowInTests: { type: "boolean", default: true },
        },
        additionalProperties: false,
      },
    ],
    messages: {
      preferFunctypeMap: "Prefer functype Map<{{keyType}}, {{valueType}}> over native Map",
      preferFunctypeMapLiteral: "Prefer Map.of(...) or Map.empty() over new Map()",
      suggestMapEmpty: "Replace with Map.empty()",
      suggestMapOf: "Replace with Map.of(...)",
      suggestMapFrom: "Replace with Map(...)",
      suggestAddImport: "Add {{symbol}} import from functype",
    },
  },

  create(context) {
    const options = context.options[0] || {}
    const allowInTests = options.allowInTests !== false
    const sourceCode = context.sourceCode

    function isInTestFile() {
      const filename = context.filename
      return (
        /\.(test|spec)\.(ts|js|tsx|jsx)$/.test(filename) ||
        filename.includes("__tests__") ||
        filename.includes("/test/") ||
        filename.includes("/tests/")
      )
    }

    function isMapImportedFromFunctype(): boolean {
      return hasFunctypeSymbol(sourceCode, "Map")
    }

    return {
      // Catch: new Map() / new Map([...]) / new Map(iterable)
      NewExpression(node: ASTNode) {
        if (allowInTests && isInTestFile()) return
        if (node.callee.type !== "Identifier" || node.callee.name !== "Map") return
        if (isMapImportedFromFunctype()) return
        if (isAlreadyUsingFunctype(node, getFunctypeImportsLegacy(context))) return

        const args = node.arguments
        const suggest: Rule.SuggestionReportDescriptor[] = []

        if (args.length === 0) {
          // new Map() -> Map.empty()
          suggest.push({
            messageId: "suggestMapEmpty",
            fix(fixer) {
              return fixer.replaceText(node, "Map.empty()")
            },
          })
        } else if (args.length === 1 && args[0].type === "ArrayExpression") {
          // new Map([[k,v], ...]) -> Map.of([k,v], ...)
          const elements = args[0].elements
            .filter((e: ASTNode) => e !== null)
            .map((e: ASTNode) => sourceCode.getText(e))
            .join(", ")
          suggest.push({
            messageId: "suggestMapOf",
            fix(fixer) {
              return fixer.replaceText(node, `Map.of(${elements})`)
            },
          })
        } else if (args.length === 1) {
          // new Map(someIterable) -> Map(someIterable)
          const argText = sourceCode.getText(args[0])
          suggest.push({
            messageId: "suggestMapFrom",
            fix(fixer) {
              return fixer.replaceText(node, `Map(${argText})`)
            },
          })
        }

        if (!isMapImportedFromFunctype()) {
          suggest.push({
            messageId: "suggestAddImport",
            data: { symbol: "Map" },
            fix: createImportFixer(sourceCode, "Map"),
          })
        }

        context.report({
          node,
          messageId: "preferFunctypeMapLiteral",
          suggest,
        })
      },

      // Catch: Map<K, V> type annotation
      TSTypeReference(node: ASTNode) {
        if (allowInTests && isInTestFile()) return
        if (!node.typeName || node.typeName.type !== "Identifier" || node.typeName.name !== "Map") return
        if (isMapImportedFromFunctype()) return

        const typeParams = node.typeParameters?.params
        const keyType = typeParams?.[0] ? sourceCode.getText(typeParams[0]) : "K"
        const valueType = typeParams?.[1] ? sourceCode.getText(typeParams[1]) : "V"

        const suggest: Rule.SuggestionReportDescriptor[] = []

        if (!isMapImportedFromFunctype()) {
          suggest.push({
            messageId: "suggestAddImport",
            data: { symbol: "Map" },
            fix: createImportFixer(sourceCode, "Map"),
          })
        }

        context.report({
          node,
          messageId: "preferFunctypeMap",
          data: { keyType, valueType },
          suggest,
        })
      },
    }
  },
}

export default rule
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd packages/plugin && pnpm vitest run tests/rules/prefer-functype-map.test.ts`
Expected: All tests PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/plugin/src/rules/prefer-functype-map.ts packages/plugin/tests/rules/prefer-functype-map.test.ts
git commit -m "feat(plugin): add prefer-functype-map rule with suggestions"
```

### Task 6: Create prefer-functype-set rule

**Files:**
- Create: `packages/plugin/src/rules/prefer-functype-set.ts`
- Create: `packages/plugin/tests/rules/prefer-functype-set.test.ts`

- [ ] **Step 1: Write tests for prefer-functype-set**

```typescript
// packages/plugin/tests/rules/prefer-functype-set.test.ts
import { describe } from "vitest"
import { ruleTester } from "../utils/rule-tester"
import rule from "../../src/rules/prefer-functype-set"

describe("prefer-functype-set", () => {
  ruleTester.run("prefer-functype-set", rule, {
    valid: [
      {
        name: "Functype Set import is allowed",
        code: 'import { Set } from "functype"\nconst s = Set.empty()',
      },
      {
        name: "Non-Set code is allowed",
        code: "const x: string = 'hello'",
      },
    ],
    invalid: [
      {
        name: "new Set() should suggest Set.empty()",
        code: "const s = new Set()",
        errors: [{
          messageId: "preferFunctypeSetLiteral",
          suggestions: [
            {
              messageId: "suggestSetEmpty",
              output: "const s = Set.empty()",
            },
          ],
        }],
      },
      {
        name: "new Set with array literal should suggest Set.of() with unwrapped elements",
        code: 'const s = new Set(["a", "b", "c"])',
        errors: [{
          messageId: "preferFunctypeSetLiteral",
          suggestions: [
            {
              messageId: "suggestSetOf",
              output: 'const s = Set.of("a", "b", "c")',
            },
          ],
        }],
      },
      {
        name: "new Set with variable should suggest Set()",
        code: "const s = new Set(items)",
        errors: [{
          messageId: "preferFunctypeSetLiteral",
          suggestions: [
            {
              messageId: "suggestSetFrom",
              output: "const s = Set(items)",
            },
          ],
        }],
      },
      {
        name: "Set type annotation should be flagged",
        code: "const s: Set<string> = new Set()",
        errors: [
          { messageId: "preferFunctypeSet" },
          { messageId: "preferFunctypeSetLiteral" },
        ],
      },
    ],
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd packages/plugin && pnpm vitest run tests/rules/prefer-functype-set.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement prefer-functype-set rule**

Same structure as `prefer-functype-map` but for `Set`. Key difference: `new Set(["a", "b"])` unwraps array elements into `Set.of("a", "b")` (variadic), whereas Map keeps the tuple structure.

```typescript
// packages/plugin/src/rules/prefer-functype-set.ts
// Same pattern as prefer-functype-map — implement with:
// - NewExpression handler: new Set() -> Set.empty(), new Set([...]) -> Set.of(...), new Set(var) -> Set(var)
// - TSTypeReference handler: Set<T> type annotation -> flag with import suggestion
// Key: Array arg unwrapping — extract individual elements from ArrayExpression for Set.of()
```

(Full implementation mirrors prefer-functype-map with s/Map/Set/g and the array unwrapping difference for `Set.of`.)

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd packages/plugin && pnpm vitest run tests/rules/prefer-functype-set.test.ts`
Expected: All tests PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/plugin/src/rules/prefer-functype-set.ts packages/plugin/tests/rules/prefer-functype-set.test.ts
git commit -m "feat(plugin): add prefer-functype-set rule with suggestions"
```

### Task 7: Register new rules in index and configs

**Files:**
- Modify: `packages/plugin/src/rules/index.ts`
- Modify: `packages/plugin/src/configs/recommended.ts`
- Modify: `packages/plugin/src/configs/strict.ts`

- [ ] **Step 1: Add to rules/index.ts**

Add imports and exports:

```typescript
import preferFunctypeMap from "./prefer-functype-map"
import preferFunctypeSet from "./prefer-functype-set"
```

Add to named exports and default export object:

```typescript
"prefer-functype-map": preferFunctypeMap,
"prefer-functype-set": preferFunctypeSet,
```

- [ ] **Step 2: Add to recommended config**

```typescript
"functype/prefer-functype-map": "warn",
"functype/prefer-functype-set": "warn",
```

- [ ] **Step 3: Add to strict config**

```typescript
"functype/prefer-functype-map": "error",
"functype/prefer-functype-set": "error",
```

- [ ] **Step 4: Run all tests**

Run: `cd packages/plugin && pnpm test`
Expected: All tests PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/plugin/src/rules/index.ts packages/plugin/src/configs/recommended.ts packages/plugin/src/configs/strict.ts
git commit -m "feat(plugin): register prefer-functype-map and prefer-functype-set in configs"
```

---

## Chunk 4: no-imperative-loops & prefer-either suggestions

### Task 8: Add suggestions to no-imperative-loops

**Files:**
- Modify: `packages/plugin/src/rules/no-imperative-loops.ts`
- Modify: `packages/plugin/tests/rules/no-imperative-loops.test.ts`

- [ ] **Step 1: Write failing suggestion tests**

Add to `invalid` array in `packages/plugin/tests/rules/no-imperative-loops.test.ts`:

```typescript
// for..of with single side effect -> forEach suggestion
{
  name: "Simple for..of should suggest forEach",
  code: `
    for (const item of items) {
      console.log(item)
    }
  `,
  errors: [{
    messageId: "noForOfLoop",
    suggestions: [
      {
        messageId: "suggestForEach",
        output: `
    items.forEach((item) => {
      console.log(item)
    })
  `,
      },
    ],
  }],
},
// for..of with single side effect (not push) -> forEach
// Note: for..of with push -> map is NOT suggested because introducing
// `const results =` changes the variable binding. The developer should
// refactor the surrounding code. Only forEach is safe to suggest for for..of.
//
// Instead, test that push patterns still get the warning but NO suggestion:
{
  name: "for..of with push should warn but not suggest (variable binding change)",
  code: `
    for (const item of items) {
      results.push(item.toUpperCase())
    }
  `,
  errors: [{
    messageId: "noForOfLoop",
    // No suggestions — push pattern requires broader refactoring
  }],
},
// for..in -> Object.keys suggestion
{
  name: "for..in should suggest Object.keys().forEach()",
  code: `
    for (const key in obj) {
      console.log(key)
    }
  `,
  errors: [{
    messageId: "noForInLoop",
    suggestions: [
      {
        messageId: "suggestObjectKeys",
        output: `
    Object.keys(obj).forEach((key) => {
      console.log(key)
    })
  `,
      },
    ],
  }],
},
// Complex body -> no suggestion
{
  name: "Complex for..of body should have no suggestion",
  code: `
    for (const item of items) {
      if (item > 0) {
        console.log(item)
      }
    }
  `,
  errors: [{
    messageId: "noForOfLoop",
    // No suggestions — complex body
  }],
},
// break/continue -> no suggestion
{
  name: "for..of with break should have no suggestion",
  code: `
    for (const item of items) {
      if (item === "stop") break
      console.log(item)
    }
  `,
  errors: [{
    messageId: "noForOfLoop",
    // No suggestions — break present
  }],
},
// destructuring -> no suggestion
{
  name: "for..of with destructuring should have no suggestion",
  code: `
    for (const { x, y } of points) {
      console.log(x, y)
    }
  `,
  errors: [{
    messageId: "noForOfLoop",
    // No suggestions — destructuring in loop variable
  }],
},
// while -> no suggestion
{
  name: "while loop should have no suggestion",
  code: `
    while (condition) {
      doSomething()
    }
  `,
  errors: [{
    messageId: "noWhileLoop",
    // No suggestions — while loops are too context-dependent
  }],
},
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd packages/plugin && pnpm vitest run tests/rules/no-imperative-loops.test.ts`
Expected: FAIL — `suggestions` not returned.

- [ ] **Step 3: Implement suggestions in no-imperative-loops**

Update `packages/plugin/src/rules/no-imperative-loops.ts`:

1. Add `hasSuggestions: true` to meta
2. Add message IDs: `suggestForEach`, `suggestObjectKeys`
3. Remove dead `getSuggestionForForLoop` function and its `void` reference
4. Add helper: `isSingleStatementBody(node)` — returns true if `node.body` is a BlockStatement with exactly 1 statement, and that statement does not contain `break`/`continue`
5. Add helper: `hasDestructuringVariable(node)` — returns true if the loop variable uses `ObjectPattern` or `ArrayPattern`

**ForOfStatement handler** — add suggest only when ALL of:
- `isSingleStatementBody(node)` is true
- `!hasDestructuringVariable(node)`
- The single statement is an ExpressionStatement (not a push)
- The iterable (`node.right`) is an Identifier or MemberExpression

```typescript
// In ForOfStatement handler, after the existing report:
const suggest: Rule.SuggestionReportDescriptor[] = []

if (isSingleStatementBody(node) && !hasDestructuringVariable(node)) {
  const varName = sourceCode.getText(node.left).replace(/^(const|let|var)\s+/, "")
  const iterableText = sourceCode.getText(node.right)
  const bodyStmt = node.body.body[0]

  // Only suggest forEach for non-push single expressions
  if (bodyStmt.type === "ExpressionStatement" && !sourceCode.getText(bodyStmt).includes(".push(")) {
    const stmtText = sourceCode.getText(bodyStmt)
    suggest.push({
      messageId: "suggestForEach",
      data: { iterable: iterableText },
      fix(fixer) {
        return fixer.replaceText(node, `${iterableText}.forEach((${varName}) => {\n  ${stmtText}\n})`)
      },
    })
  }
}

context.report({ node, messageId: "noForOfLoop", suggest: suggest.length > 0 ? suggest : undefined })
```

**ForInStatement handler** — suggest `Object.keys(obj).forEach(...)` when body is single statement:

```typescript
if (isSingleStatementBody(node)) {
  const keyVar = sourceCode.getText(node.left).replace(/^(const|let|var)\s+/, "")
  const objText = sourceCode.getText(node.right)
  const stmtText = sourceCode.getText(node.body.body[0])
  suggest.push({
    messageId: "suggestObjectKeys",
    data: { object: objText },
    fix(fixer) {
      return fixer.replaceText(node, `Object.keys(${objText}).forEach((${keyVar}) => {\n  ${stmtText}\n})`)
    },
  })
}
```

6. `ForStatement`, `WhileStatement`, `DoWhileStatement` — never add suggestions, just report

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd packages/plugin && pnpm vitest run tests/rules/no-imperative-loops.test.ts`
Expected: All tests PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/plugin/src/rules/no-imperative-loops.ts packages/plugin/tests/rules/no-imperative-loops.test.ts
git commit -m "feat(plugin): add suggestions to no-imperative-loops rule"
```

### Task 9: Add suggestions to prefer-either

**Files:**
- Modify: `packages/plugin/src/rules/prefer-either.ts`
- Modify: `packages/plugin/tests/rules/prefer-either.test.ts`

- [ ] **Step 1: Write failing suggestion tests**

Add to `invalid` array in `packages/plugin/tests/rules/prefer-either.test.ts`:

```typescript
// Simple try/catch with single expression and empty catch -> Try() suggestion
{
  name: "Simple try/catch with empty catch should suggest Try()",
  code: `
    function parse(json: string) {
      try {
        return JSON.parse(json)
      } catch (e) {}
    }
  `,
  errors: [{
    messageId: "preferEitherOverTryCatch",
    suggestions: [
      {
        messageId: "suggestTry",
        output: `
    function parse(json: string) {
      return Try(() => JSON.parse(json))
    }
  `,
      },
    ],
  }],
},
// Async try/catch -> Try.fromPromise() suggestion
{
  name: "Async try/catch should suggest Try.fromPromise()",
  code: `
    async function fetchData(url: string) {
      try {
        return await fetch(url)
      } catch (e) {}
    }
  `,
  errors: [{
    messageId: "preferEitherOverTryCatch",
    suggestions: [
      {
        messageId: "suggestTryFromPromise",
        output: `
    async function fetchData(url: string) {
      return Try.fromPromise(fetch(url))
    }
  `,
      },
    ],
  }],
},
// throw new Error -> Either.left() suggestion
{
  name: "throw in function body should suggest Either.left()",
  code: `
    function validateAge(age: number) {
      if (age < 0) {
        throw new Error('Age cannot be negative')
      }
      return age
    }
  `,
  errors: [
    {
      messageId: "preferEitherReturn",
    },
    {
      messageId: "preferEitherOverThrow",
      suggestions: [
        {
          messageId: "suggestEitherLeft",
        },
      ],
    },
  ],
},
// Multi-statement try body -> no suggestion
{
  name: "Multi-statement try body should have no suggestion",
  code: `
    function complex() {
      try {
        const a = parse()
        const b = transform(a)
        return b
      } catch (e) {
        return null
      }
    }
  `,
  errors: [{
    messageId: "preferEitherOverTryCatch",
    // No suggestions — multi-statement body
  }],
},
// Multi-statement catch -> no suggestion
{
  name: "Multi-statement catch should have no suggestion",
  code: `
    function risky() {
      try {
        return doSomething()
      } catch (e) {
        console.error(e)
        notifyAdmin(e)
        return null
      }
    }
  `,
  errors: [{
    messageId: "preferEitherOverTryCatch",
    // No suggestions — complex catch
  }],
},
// throw inside loop -> no suggestion (return would change control flow)
{
  name: "throw inside loop should have no suggestion",
  code: `
    function validateItems(items: string[]) {
      items.forEach(item => {
        if (!item) {
          throw new Error('Invalid item')
        }
      })
    }
  `,
  errors: [{
    messageId: "preferEitherOverThrow",
    // No suggestions — throw is inside a callback, not direct function body
  }],
},
// throw string literal -> Either.left(new Error(String(...)))
{
  name: "throw string literal should suggest Either.left with Error wrapping",
  code: `
    function fail() {
      throw "something went wrong"
    }
  `,
  errors: [
    {
      messageId: "preferEitherOverThrow",
      suggestions: [
        {
          messageId: "suggestEitherLeft",
        },
      ],
    },
  ],
},
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd packages/plugin && pnpm vitest run tests/rules/prefer-either.test.ts`
Expected: FAIL — `suggestions` not returned.

- [ ] **Step 3: Implement suggestions in prefer-either**

Update `packages/plugin/src/rules/prefer-either.ts`:

1. Add `hasSuggestions: true` to meta
2. Add message IDs: `suggestTry`, `suggestTryFromPromise`, `suggestEitherLeft`, `suggestAddImport`
3. Import `createImportFixer`, `hasFunctypeSymbol`

**TryStatement handler** — add helpers and suggest logic:

```typescript
function isSimpleTryBody(node: ASTNode): boolean {
  // Try block must have exactly 1 statement
  return node.block?.body?.length === 1
}

function isSimpleCatch(node: ASTNode): boolean {
  if (!node.handler?.body) return false
  const catchBody = node.handler.body.body
  // Empty catch, single return, or single expression
  return catchBody.length === 0 || (catchBody.length === 1 &&
    (catchBody[0].type === "ReturnStatement" || catchBody[0].type === "ExpressionStatement"))
}

function tryBodyHasAwait(node: ASTNode): boolean {
  const stmt = node.block.body[0]
  if (stmt.type === "ReturnStatement" && stmt.argument?.type === "AwaitExpression") return true
  if (stmt.type === "ExpressionStatement" && stmt.expression?.type === "AwaitExpression") return true
  return false
}
```

In TryStatement, after the existing re-throw check:
```typescript
const suggest: Rule.SuggestionReportDescriptor[] = []

if (isSimpleTryBody(node) && isSimpleCatch(node)) {
  const tryStmt = node.block.body[0]
  const isReturn = tryStmt.type === "ReturnStatement"
  const expr = isReturn ? tryStmt.argument : tryStmt.expression

  if (tryBodyHasAwait(node)) {
    // Unwrap the await — Try.fromPromise takes the Promise directly
    const awaitExpr = expr.type === "AwaitExpression" ? expr.argument : expr
    const exprText = sourceCode.getText(awaitExpr)
    suggest.push({
      messageId: "suggestTryFromPromise",
      fix(fixer) {
        const replacement = isReturn ? `return Try.fromPromise(${exprText})` : `Try.fromPromise(${exprText})`
        return fixer.replaceText(node, replacement)
      },
    })
  } else {
    const exprText = sourceCode.getText(expr)
    suggest.push({
      messageId: "suggestTry",
      fix(fixer) {
        const replacement = isReturn ? `return Try(() => ${exprText})` : `Try(() => ${exprText})`
        return fixer.replaceText(node, replacement)
      },
    })
  }

  // Add import suggestion
  const symbol = tryBodyHasAwait(node) ? "Try" : "Try"
  if (!hasFunctypeSymbol(sourceCode, symbol)) {
    suggest.push({ messageId: "suggestAddImport", data: { symbol }, fix: createImportFixer(sourceCode, symbol) })
  }
}

context.report({ node, messageId: "preferEitherOverTryCatch", suggest: suggest.length > 0 ? suggest : undefined })
```

**ThrowStatement handler** — add `isDirectInFunctionBody` check:

```typescript
function isDirectInFunctionBody(node: ASTNode): boolean {
  const parent = node.parent
  if (!parent) return false
  // Direct child of function body
  if (parent.type === "BlockStatement" && isFunctionLike(parent.parent)) return true
  // Direct child of if-block in function body
  if (parent.type === "BlockStatement" && parent.parent?.type === "IfStatement") {
    const ifParent = parent.parent.parent
    return ifParent?.type === "BlockStatement" && isFunctionLike(ifParent.parent)
  }
  return false
}

function isFunctionLike(node: ASTNode): boolean {
  return ["FunctionDeclaration", "FunctionExpression", "ArrowFunctionExpression", "MethodDefinition"].includes(node?.type)
}
```

In ThrowStatement, add suggest when direct in function body:
```typescript
const suggest: Rule.SuggestionReportDescriptor[] = []

if (isDirectInFunctionBody(node)) {
  const throwArg = node.argument
  const argText = sourceCode.getText(throwArg)
  // Wrap non-Error values in new Error(String(...))
  const isErrorExpr = throwArg?.type === "NewExpression" && throwArg.callee?.name === "Error"
  const eitherArg = isErrorExpr ? argText : `new Error(String(${argText}))`

  suggest.push({
    messageId: "suggestEitherLeft",
    fix(fixer) {
      return fixer.replaceText(node, `return Either.left(${eitherArg})`)
    },
  })

  if (!hasFunctypeSymbol(sourceCode, "Either")) {
    suggest.push({ messageId: "suggestAddImport", data: { symbol: "Either" }, fix: createImportFixer(sourceCode, "Either") })
  }
}

context.report({ node, messageId: "preferEitherOverThrow", suggest: suggest.length > 0 ? suggest : undefined })
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd packages/plugin && pnpm vitest run tests/rules/prefer-either.test.ts`
Expected: All tests PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/plugin/src/rules/prefer-either.ts packages/plugin/tests/rules/prefer-either.test.ts
git commit -m "feat(plugin): add suggestions to prefer-either rule"
```

---

## Chunk 5: Final validation

### Task 10: Full validation

**Files:** None (validation only)

- [ ] **Step 1: Run full test suite**

Run: `cd packages/plugin && pnpm test`
Expected: All tests pass (existing 116 + ~40 new).

- [ ] **Step 2: Run full validate pipeline**

Run: `pnpm validate`
Expected: Both packages pass format, lint, typecheck, test, and build.

- [ ] **Step 3: Fix any issues**

If any step fails, fix and re-run. Common issues:
- Import ordering (run `pnpm format` to fix)
- Lint errors in new code (run `pnpm lint` to identify)
- Type errors (run `pnpm typecheck` to identify)

- [ ] **Step 4: Final commit if any fixes needed**

```bash
git add -A
git commit -m "fix(plugin): resolve validation issues"
```
