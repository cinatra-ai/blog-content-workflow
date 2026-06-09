<!-- refreshed: 2026-06-09 -->
# Architecture

**Analysis Date:** 2026-06-09

## System Overview

```text
┌──────────────────────────────────────────────────────────────┐
│              Cinatra Platform (host runtime)                  │
│   Parses & compiles the extension at install/publish time    │
└──────────┬───────────────────────┬───────────────────────────┘
           │                       │
           ▼                       ▼
┌────────────────────┐  ┌──────────────────────────────────────┐
│  Workflow Engine   │  │        Operator Dashboard             │
│  cinatra/          │  │        cinatra/dashboard.json         │
│  workflow.bpmn     │  │  (portlet composition, wired inputs)  │
└────────────────────┘  └──────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────┐
│              External Agents / Services                       │
│  @cinatra-ai/blog-wordpress-publish-agent  (serviceTask)     │
│  WordPress MCP connector  (wordpressInstanceId)              │
└──────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| BPMN workflow definition | Declares the approval-gated publish DAG; compiled to a WorkflowSpec by the platform at install time | `cinatra/workflow.bpmn` |
| Dashboard composition | Declares the operator portlet layout and inter-portlet wiring for the blog workspace | `cinatra/dashboard.json` |
| Package manifest | Registers the extension as `kind:"workflow"` with the Cinatra marketplace; declares workflowVersion and zero runtime dependencies | `package.json` |
| TypeScript entry point | No-op module boundary; exists solely to satisfy the TypeScript module contract; carries the authoritative doc comment for the extension | `src/index.ts` |
| CI kind gate | Zero-dependency Node.js script; validates package shape and BPMN well-formedness in CI without requiring the @cinatra-ai registry | `extension-kind-gate.mjs` |

## Pattern Overview

**Overall:** Declarative sidecar extension — no application runtime code. All behaviour is expressed as data (BPMN XML + JSON portlet declarations) parsed by the Cinatra host platform.

**Key Characteristics:**
- Zero runtime code surface: `src/index.ts` exports nothing; the platform reads the sidecars directly
- The workflow DAG is authored once in `cinatra/workflow.bpmn` and compiled marketplace-side; this repo ships only the source definition
- The dashboard is a declarative portlet graph (`cinatra/dashboard.json`) with typed input/output wiring between portlet instances; the platform renders it
- All external integrations (WordPress, blog agents) are referenced by package name or instance ID in sidecar data, never imported as code

## Layers

**Sidecar data layer:**
- Purpose: Declares the extension's behaviour as static artifacts consumed by the Cinatra platform
- Location: `cinatra/`
- Contains: `workflow.bpmn` (BPMN 2.0 + Cinatra Profile 1.0 extensions), `dashboard.json` (portlet composition)
- Depends on: Platform BPMN compiler and dashboard renderer (host)
- Used by: Cinatra platform at install/publish and at runtime

**Package boundary layer:**
- Purpose: Identifies the extension to the marketplace and enforces the `kind:"workflow"` contract
- Location: `package.json`, `src/index.ts`
- Contains: Package metadata, `cinatra.*` manifest fields, empty TS export
- Depends on: Nothing (zero runtime dependencies)
- Used by: npm/pnpm publish pipeline, marketplace registry

**CI validation layer:**
- Purpose: Lightweight pre-publish sanity gate; catches wrong kind, missing/malformed BPMN, and retired CRM primitives before the marketplace compiler runs
- Location: `extension-kind-gate.mjs`, `.github/workflows/ci.yml`, `.github/workflows/release.yml`
- Contains: Self-contained Node.js validators (`validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `validateAgent`)
- Depends on: Node.js builtins only (no `@cinatra-ai/*`)
- Used by: GitHub Actions CI on every push/PR to `main`

## Data Flow

### Blog Post Publish Path (runtime, orchestrated by platform)

1. Operator opens the dashboard — platform renders portlets from `cinatra/dashboard.json`
2. Operator selects project → idea → post via the `projects-pane` → `ideas-pane` → `posts-pane` portlets (output `selectedId` wired as `parentId` input downstream)
3. Operator edits post body in `draft-editor` portlet (`artifact-edit-text`); each save mints a new artifact and swaps `postArtifactId` on the blog-post object; `version-history` portlet reflects changes
4. Operator previews/regenerates hero image via `hero-image` portlet (`artifact-edit-binary-prompt`, `refSwapMode:"auto"`)
5. Operator clicks `publish-launcher` portlet (`workflow-launcher`); `projectId` and `postId` are forwarded from upstream portlet outputs; `wordpressInstanceId` is picked in the launcher
6. Platform instantiates the `blog-content-workflow` template, running the BPMN DAG:
   a. `start` → `review_publish_bundle` (approval userTask, organization-level, rejection policy: needs_revision)
   b. On approval → `create_wordpress_draft` (serviceTask invoking `@cinatra-ai/blog-wordpress-publish-agent` with `{projectId, postId, wordpressInstanceId}`)
   c. On draft success → `publish_in_wordpress_admin` (manualTask — operator publishes in WP admin UI)
   d. → `notify_publish_checkpoint_complete` (sendTask — emits "Blog post {{postId}} published to {{wordpressInstanceId}}")
   e. → `end`
7. `publish-status` portlet (`workflow-status`) polls/streams the running workflow's task and status updates

### CI Gate Path

1. Push to `main` or PR opened → `.github/workflows/ci.yml` triggers
2. `build` job: classifies repo as source-mirror (has `@cinatra-ai/*` optional peers) or standalone; skips install/typecheck/test for source mirrors; runs `npm pack --dry-run`
3. `kind-gates` job (after `build`): runs `node extension-kind-gate.mjs --package-root .`; validates package shape and BPMN sanity; exits 0 or 1

**State Management:**
- No client-side state. All workflow state (task status, artifact versions) is owned by the Cinatra platform backend and surfaced to the dashboard via portlet inputs/outputs.

## Key Abstractions

**BPMN Placeholders:**
- Purpose: Typed runtime parameters (`projectId`, `postId`, `wordpressInstanceId`) declared in the BPMN `extensionElements` so the platform can render typed pickers in the `workflow-launcher` portlet and inject values into task inputs
- Examples: `cinatra/workflow.bpmn` lines 11–21
- Pattern: `<cinatra:placeholder name="..." type="string" required="true">` with a `<cinatra:placeholderHint kind="...">` for UI picker binding

**Portlet wiring:**
- Purpose: Inter-portlet data flow is declared declaratively via `inputs.fromInstanceId` / `outputs` in `cinatra/dashboard.json`; the platform resolves values reactively
- Examples: `cinatra/dashboard.json` — `ideas-pane.inputs.parentId` receives `projects-pane.selectedId`
- Pattern: `{ "fromInstanceId": "<instanceId>", "key": "<outputKey>" }`

**Cinatra BPMN task kinds:**
- `cinatra:taskKind value="approval"` — maps to a platform-managed approval UI
- `cinatra:agentRef package="..."` — delegates a serviceTask to a named agent extension
- `bpmn:manualTask` — operator action outside the platform (e.g., clicking Publish in WP admin)
- `bpmn:sendTask` + `cinatra:messageBody` — platform-delivered notification

## Entry Points

**Platform install/compile:**
- Location: `cinatra/workflow.bpmn`
- Triggers: `parseWorkflowBpmnSidecar` called by the Cinatra marketplace at extension install or publish
- Responsibilities: Provides the DAG definition compiled to a WorkflowSpec

**Dashboard render:**
- Location: `cinatra/dashboard.json`
- Triggers: Platform loads the dashboard when an operator opens the blog workspace
- Responsibilities: Declares all portlets, their kinds/versions/slots, and their wiring

**CI gate:**
- Location: `extension-kind-gate.mjs` (invoked as `node extension-kind-gate.mjs --package-root .`)
- Triggers: GitHub Actions `kind-gates` job on every push/PR
- Responsibilities: Validates package.json shape and BPMN well-formedness; exits non-zero on violations

## Architectural Constraints

- **No runtime code:** The extension ships zero runtime JavaScript/TypeScript beyond the empty `src/index.ts` export. All behaviour is declarative.
- **No @cinatra-ai/* runtime dependencies:** `package.json` declares zero `dependencies` or `devDependencies`. Any first-party packages would be optional `peerDependencies` resolved by the monorepo at build time.
- **Single BPMN sidecar:** Exactly one `cinatra/workflow.bpmn` is allowed. The gate (`findWorkflowSidecars`) enforces this; duplicates cause a CI failure.
- **Inline workflow forbidden:** The `cinatra.workflow` key in `package.json` is explicitly prohibited; the BPMN sidecar is the only permitted workflow declaration.
- **Idempotent publish:** Re-launching with identical `{projectId, postId, wordpressInstanceId}` returns a structured idempotent-noop response instead of creating a duplicate WordPress draft (behaviour owned by the platform/agent, not this repo).

## Anti-Patterns

### Declaring workflow inline in package.json

**What happens:** Setting `cinatra.workflow` in `package.json` instead of providing `cinatra/workflow.bpmn`
**Why it's wrong:** The gate (`validateWorkflowPackageShape`) rejects it; the marketplace parser requires the BPMN sidecar
**Do this instead:** Author the DAG in `cinatra/workflow.bpmn` only

### Adding first-party deps to dependencies/devDependencies

**What happens:** Placing `@cinatra-ai/*` packages in `dependencies` or `devDependencies`
**Why it's wrong:** These packages are not published to any registry; CI would fail at install on the extracted standalone repo
**Do this instead:** Declare them as `peerDependencies` with `peerDependenciesMeta.<pkg>.optional: true`

## Error Handling

**Strategy:** Fail-fast in CI via `extension-kind-gate.mjs`. All violation messages are collected into a `string[]` and printed as a bulleted list before `process.exit(1)`.

**Patterns:**
- Pure validator functions (`validateWorkflowPackageShape`, `validateBpmnSanity`) return `string[]` errors — never throw
- IO-layer functions (`validateWorkflow`, `validateAgent`) catch read/parse errors and push them into the same errors array
- The `main()` dispatcher exits 0 for no violations, 1 for any violations

## Cross-Cutting Concerns

**Logging:** CI gate prints to `console.log` (pass) or `console.error` (violations). No application-level logging.
**Validation:** Performed exclusively at CI time by `extension-kind-gate.mjs` and authoritatively marketplace-side at publish/install.
**Authentication:** Credentials never reach the client. `wordpressInstanceId` is a reference resolved by the WordPress MCP connector; secrets stay server-side.

---

*Architecture analysis: 2026-06-09*
