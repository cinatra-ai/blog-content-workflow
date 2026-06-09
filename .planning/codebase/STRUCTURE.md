# Codebase Structure

**Analysis Date:** 2026-06-09

## Directory Layout

```
blog-content-workflow/
├── cinatra/                  # Cinatra sidecar artifacts (consumed by platform)
│   ├── workflow.bpmn         # BPMN 2.0 + Cinatra Profile 1.0 workflow definition
│   └── dashboard.json        # Operator dashboard portlet composition
├── src/
│   └── index.ts              # No-op TS module boundary; carries extension doc comment
├── .github/
│   └── workflows/
│       ├── ci.yml            # Standalone CI: classify, typecheck, pack dry-run, kind gate
│       └── release.yml       # Release pipeline
├── extension-kind-gate.mjs   # Zero-dependency CI kind gate (Node builtins only)
├── package.json              # Package manifest + cinatra extension metadata
├── tsconfig.json             # TypeScript config (minimal; src/index.ts only)
├── .npmrc                    # npm registry config
├── LICENSE                   # Apache-2.0
└── README.md                 # Extension overview and capability summary
```

## Directory Purposes

**`cinatra/`:**
- Purpose: Hosts all Cinatra platform sidecar files — the only files the platform parser reads at install/publish time
- Contains: `workflow.bpmn` (BPMN process definition), `dashboard.json` (portlet layout)
- Key files: `cinatra/workflow.bpmn`, `cinatra/dashboard.json`

**`src/`:**
- Purpose: TypeScript package boundary; required by the `"main"` and `"types"` fields in `package.json`
- Contains: A single `index.ts` with an empty `export {}` and the authoritative extension doc comment
- Key files: `src/index.ts`

**`.github/workflows/`:**
- Purpose: GitHub Actions CI/CD pipelines for the standalone extracted repo
- Contains: `ci.yml` (build + kind gate), `release.yml` (publish)
- Key files: `.github/workflows/ci.yml`, `.github/workflows/release.yml`

## Key File Locations

**Entry Points (platform):**
- `cinatra/workflow.bpmn`: BPMN DAG compiled to WorkflowSpec at install time
- `cinatra/dashboard.json`: Dashboard portlet graph loaded when operator opens the blog workspace

**Extension Manifest:**
- `package.json`: Declares `name`, `version`, `cinatra.kind`, `cinatra.workflowVersion`, `cinatra.dependencies`

**CI Gate:**
- `extension-kind-gate.mjs`: Self-contained validator; run as `node extension-kind-gate.mjs --package-root .`

**Type Boundary:**
- `src/index.ts`: Empty TS export satisfying `"main"` / `"types"` in `package.json`

**Build Config:**
- `tsconfig.json`: TypeScript compiler config
- `.npmrc`: Registry settings (existence noted; contents not read)

## Naming Conventions

**Files:**
- Sidecar artifacts: lowercase with extension matching platform contract — `workflow.bpmn`, `dashboard.json`
- CI gate script: kebab-case `.mjs` — `extension-kind-gate.mjs`
- TypeScript sources: `index.ts` (single entry point)

**Directories:**
- Platform sidecars: `cinatra/` (fixed name required by platform)
- TypeScript sources: `src/` (conventional)

**Package name pattern:**
- Must match `@<scope>/<slug>-workflow` — validated by `extension-kind-gate.mjs` at CI time

**BPMN element IDs:**
- snake_case for task/event/flow IDs: `review_publish_bundle`, `create_wordpress_draft`, `flow_start_review`

**Dashboard portlet instance IDs:**
- kebab-case: `projects-pane`, `draft-editor`, `publish-launcher`, `publish-status`

## Where to Add New Code

**New workflow task (BPMN):**
- Add `<bpmn:*Task>` element and sequence flows in `cinatra/workflow.bpmn`
- Use `<cinatra:agentRef>` for serviceTask agent delegation
- Keep element IDs in snake_case

**New dashboard portlet:**
- Add an entry to the `portlets` array in `cinatra/dashboard.json`
- Wire inputs via `fromInstanceId` / `key` referencing an existing portlet's `outputs`
- Choose a kebab-case `instanceId`

**New workflow placeholder (runtime parameter):**
- Add `<cinatra:placeholder>` inside `<cinatra:placeholders>` in `cinatra/workflow.bpmn`
- Add a matching `<cinatra:placeholderHint kind="...">` for the launcher UI picker

**Documentation / extension narrative:**
- Update `README.md` for capability changes
- Update the doc comment in `src/index.ts` for architectural changes

## Special Directories

**`cinatra/`:**
- Purpose: Platform-consumed sidecar directory; name is fixed by the Cinatra extension contract
- Generated: No (hand-authored)
- Committed: Yes

**`.github/`:**
- Purpose: GitHub Actions workflow definitions injected by the monorepo extraction script
- Generated: Partially (base `ci.yml` is a template; kind-specific gate steps are appended by the extraction script)
- Committed: Yes

---

*Structure analysis: 2026-06-09*
