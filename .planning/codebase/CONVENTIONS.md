# Coding Conventions

**Analysis Date:** 2026-06-09

## Repository Type

This is a Cinatra Marketplace **workflow extension** — a content-only package. The only runtime TypeScript file (`src/index.ts`) exports nothing; the extension surface is declared in `cinatra/workflow.bpmn` (BPMN Profile 1.0 sidecar) and `cinatra/dashboard.json` (operator portlet composition). All executable logic lives in `extension-kind-gate.mjs`, a self-contained CI gate written in plain ESM JavaScript (no TypeScript, no dependencies beyond Node builtins).

## Naming Patterns

**Files:**
- Kebab-case for all filenames: `extension-kind-gate.mjs`, `workflow.bpmn`, `dashboard.json`
- Package name follows the enforced pattern `@<scope>/<slug>-workflow` (validated in gate): `@cinatra-ai/blog-content-workflow`
- CI/workflow files use descriptive lowercase names: `ci.yml`, `release.yml`

**Functions (in `extension-kind-gate.mjs`):**
- camelCase for all exported and internal functions: `parseArgs`, `validateAgent`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `validateWorkflow`, `runGate`
- Helper/utility functions follow a `verb + Noun` pattern: `walkLlmStrings`, `scanOasString`, `prefixOf`, `localOf`, `wordBoundary`
- Entry point named `main()` following Node convention

**Variables:**
- camelCase for local variables and parameters: `packageRoot`, `bpmnPrefixes`, `openTags`, `allSidecars`
- SCREAMING_SNAKE_CASE for module-level constants: `LLM_VISIBLE_FIELDS`, `BANNED_PRIMITIVES`, `BANNED_TYPEHINTS`, `PRIMITIVE_PATTERNS`, `BPMN_MODEL_NS`, `WORKFLOW_PACKAGE_NAME_RE`, `OBJECTS_LIST_CRM_RE`

**Types/Patterns:**
- `Set` for O(1) membership checks (e.g. `LLM_VISIBLE_FIELDS`, `bpmnPrefixes`, `SKIP`)
- Array of plain objects for structured findings: `{ field, token, reason }`

## Code Style

**Formatting:**
- No formatter config detected (no `.prettierrc`, `.editorconfig`, `biome.json`)
- Consistent 2-space indentation in `extension-kind-gate.mjs`
- Double quotes for strings throughout

**Linting:**
- No ESLint or Biome config detected

**TypeScript Config (`tsconfig.json`):**
- `strict: true` with `noImplicitAny: false` (strict mode with one relaxation)
- `verbatimModuleSyntax: true` — import type must use `import type`
- `isolatedModules: true`
- Target: ES2023, module: ESNext, moduleResolution: bundler
- `src/index.ts` is the only TypeScript source; it is a no-op re-export stub

## Import Organization

**In `extension-kind-gate.mjs`:**
- All imports at file top, grouped as a single block of Node built-in named imports
- `import { readFileSync, existsSync, readdirSync } from "node:fs";`
- `import { resolve, join, basename, dirname, relative } from "node:path";`
- Uses `node:` protocol prefix for all built-in imports (explicit Node.js namespace)
- Zero external or @cinatra-ai imports — enforced by design (self-contained CI gate)

## Error Handling

**Pattern:** Pure functions return `string[]` errors arrays — never throw, never console.error inside logic functions. Only the top-level `main()` writes to stderr and calls `process.exit`.

**Example:**
```js
export function validateBpmnSanity(xml) {
  const errors = [];
  if (typeof xml !== "string" || xml.trim() === "") {
    errors.push("cinatra/workflow.bpmn is empty");
    return errors;  // early return on fatal condition
  }
  // ... checks push to errors
  return errors;
}
```

**Try/catch:** Used only at I/O boundaries (file reads). Errors are caught, formatted, and pushed into the errors array — never re-thrown.

## Logging

**Framework:** `console.log` / `console.error` only in `main()` — nowhere else.

**Patterns:**
- Success: `console.log("✓ extension-kind-gate: ...")` → stdout
- Failure: `console.error("✗ extension-kind-gate: ...")` → stderr with bullet list of violations

## Comments

**When to Comment:**
- Section dividers with `// ---...---` banners for logical groupings within `extension-kind-gate.mjs`
- Long block comments explain WHY a design decision was made (especially why the gate is self-contained and zero-dependency)
- Inline `//` comments on regex patterns explaining their purpose
- `tsconfig.json` uses a `"//"` key as a prose comment (JSON doesn't support `//` natively)

**JSDoc/TSDoc:** Not used — no TypeScript functions with public API surface (the only `.ts` file exports nothing callable).

## Function Design

**Size:** Functions are focused and short-to-medium. `validateBpmnSanity` is the largest (~80 lines) because it encodes a tag-balance XML walk.

**Parameters:** Single `packageRoot: string` parameter for all top-level validators. Pure helper functions take the minimal set of arguments needed.

**Return Values:** Consistently `string[]` (errors) for all validators. `runGate` returns `{ kind, errors }`. `parseArgs` returns `{ packageRoot }`.

## Module Design

**Exports:** Named exports for all testable units: `parseArgs`, `validateAgent`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `validateWorkflow`, `runGate`. The `main()` function is NOT exported.

**Direct invocation guard:** The file uses a `invokedDirectly` check against `import.meta.url` to call `main()` only when run as a script, allowing all exports to be imported cleanly in tests.

**`src/index.ts`:** Single `export {};` — the extension has no TypeScript runtime surface. The package's `"main"` and `"types"` both point here for marketplace tooling compatibility.

## Package Conventions

**`package.json` required shape** (enforced by gate and CI):
- `cinatra.kind: "workflow"`
- `cinatra.apiVersion: "cinatra.ai/v1"`
- `cinatra.workflowVersion`: positive integer
- `cinatra.dependencies`: array (may be empty)
- No first-party `@cinatra-ai/*` packages in `dependencies`, `devDependencies`, or `optionalDependencies`
- First-party peers must be `peerDependencies` with `peerDependenciesMeta[pkg].optional: true`
- No committed lockfile — `pnpm install --no-frozen-lockfile` in CI

---

*Convention analysis: 2026-06-09*
