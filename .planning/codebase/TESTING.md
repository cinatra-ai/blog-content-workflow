# Testing Patterns

**Analysis Date:** 2026-06-09

## Test Framework

**Runner:** Not detected — no `jest.config.*`, `vitest.config.*`, `mocha`, or similar config file present in the repository.

**Assertion Library:** Not detected.

**Run Commands:**
```bash
# package.json has no "test" script defined.
# CI runs: corepack pnpm test --if-present (exits 0 if absent)
```

## Test File Organization

**Location:** No test files (`.test.*` or `.spec.*`) detected in the repository.

**Note:** This is a **content-only workflow extension**. The only executable logic is `extension-kind-gate.mjs`, which is a self-contained CI gate. The gate is tested indirectly through CI (`ci.yml`) via the `node extension-kind-gate.mjs --package-root .` step — it runs against this repo's own `package.json` and `cinatra/workflow.bpmn` as a live integration check.

## Validation via CI (Substitute for Unit Tests)

The repository's quality assurance relies entirely on CI gates rather than in-repo test suites:

**`build` job (`.github/workflows/ci.yml`):**
1. Classifies the repo as a source mirror (has `@cinatra-ai/*` optional peers) or standalone
2. For source mirrors: skips install, typecheck, and test — delegates to the cinatra monorepo
3. For standalone repos: runs `pnpm install`, `tsc --noEmit`, and `pnpm test --if-present`
4. Runs `npm pack --dry-run` to validate package shape and publish payload on every build

**`kind-gates` job (`.github/workflows/ci.yml`):**
- Runs after `build` (declared `needs: build`)
- Executes `node extension-kind-gate.mjs --package-root .`
- Validates: `package.json` shape (name pattern, `cinatra.kind`, `cinatra.workflowVersion`, allowed `cinatra.*` keys, no inline `cinatra.workflow`), presence and well-formedness of `cinatra/workflow.bpmn`, BPMN namespace and root element correctness, at least one `<bpmn:process>` element

## Gate Logic Architecture (Testable Exports)

`extension-kind-gate.mjs` exports all validator functions as named exports, making them importable by an external test suite:

| Export | What it validates |
|--------|------------------|
| `parseArgs(argv)` | CLI argument parsing |
| `validateAgent(packageRoot)` | OAS JSON retired-primitive scan |
| `validateWorkflowPackageShape(pkg)` | `package.json` cinatra shape rules |
| `validateBpmnSanity(xml)` | XML well-formedness + BPMN profile checks |
| `findWorkflowSidecars(packageRoot)` | Recursive sidecar discovery |
| `validateWorkflow(packageRoot)` | Composite workflow gate |
| `runGate(packageRoot)` | Top-level dispatcher returning `{ kind, errors }` |

All functions are **pure or near-pure** (only I/O in file-read calls) and return `string[]` errors — straightforward to unit test without mocking complex infrastructure.

## Mocking

**Framework:** Not applicable — no test suite exists.

**If adding tests:** The pure-function design means most validators need no mocking. Only file-system-dependent functions (`validateWorkflow`, `findWorkflowSidecars`, `validateAgent`, `runGate`) require either a real temp directory or `fs` module mocking.

## Fixtures and Factories

**Test Data:** Not applicable — no test suite exists.

**Natural fixture inputs for future tests:**
- Valid/invalid `package.json` objects for `validateWorkflowPackageShape`
- Valid/truncated/non-BPMN XML strings for `validateBpmnSanity`
- The real `cinatra/workflow.bpmn` from this repo as a golden fixture

## Coverage

**Requirements:** None enforced — no coverage tooling configured.

**Current state:** The gate logic in `extension-kind-gate.mjs` is covered only by the live CI run against this repo's own artifacts.

## Test Types

**Unit Tests:** Not present. The gate exports are architected for unit testing but no tests are written.

**Integration Tests:** The CI `kind-gates` job acts as a live integration test — it runs the gate against real repo files on every push and PR.

**E2E Tests:** Not applicable.

## Adding Tests (Guidance)

To add a test suite to this repo:

1. Add a test runner to `package.json` devDependencies (e.g., `vitest` or `node:test` built-in — preferred for zero-dependency alignment with the gate's philosophy)
2. Add a `"test"` script to `package.json`
3. Place test files alongside or in a `test/` directory — e.g., `test/extension-kind-gate.test.mjs`
4. Import named exports directly: `import { validateBpmnSanity, validateWorkflowPackageShape } from "../extension-kind-gate.mjs"`
5. Focus on `validateBpmnSanity` and `validateWorkflowPackageShape` first — they are pure string/object functions with no I/O

**Example structure (using `node:test`):**
```js
import { test } from "node:test";
import assert from "node:assert/strict";
import { validateWorkflowPackageShape, validateBpmnSanity } from "../extension-kind-gate.mjs";

test("valid workflow package passes", () => {
  const errors = validateWorkflowPackageShape({
    name: "@cinatra-ai/blog-content-workflow",
    cinatra: { kind: "workflow", apiVersion: "cinatra.ai/v1", workflowVersion: 1, dependencies: [] },
  });
  assert.deepEqual(errors, []);
});
```

---

*Testing analysis: 2026-06-09*
