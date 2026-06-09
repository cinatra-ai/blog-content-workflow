# External Integrations

**Analysis Date:** 2026-06-09

## APIs & External Services

**WordPress:**
- WordPress (self-hosted or cloud instances) - target CMS for blog draft creation and publishing
  - SDK/Client: `@cinatra-ai/blog-wordpress-publish-agent` (agent referenced in BPMN via `cinatra:agentRef`)
  - Auth: Credentials managed by the Cinatra platform's WordPress MCP connector; identified by `wordpressInstanceId` placeholder — credentials never reach the client
  - Interaction: The `create_wordpress_draft` BPMN service task delegates to the Blog WordPress Publish agent, passing `projectId`, `postId`, and `wordpressInstanceId` as task inputs

**Cinatra Marketplace / Registry:**
- `registry.cinatra.ai` - target registry for publishing this extension
  - Auth: `CINATRA_MARKETPLACE_VENDOR_TOKEN` (org secret, consumed by the reusable release workflow at `cinatra-ai/.github`)

## Data Storage

**Databases:**
- Not applicable — this extension holds no database client or connection. All data (blog projects, ideas, posts, artifacts) is managed by the Cinatra platform's object store, accessed via dashboard portlet primitives (`object-list`, `object-detail`, `artifact-edit-text`, `artifact-edit-binary-prompt`, `artifact-version-history`)

**File Storage:**
- Not applicable — binary/image artifacts are managed by the Cinatra platform via the `hero-image` portlet (`artifact-edit-binary-prompt` kind with `refSwapMode: "auto"`)

**Caching:**
- None

## Authentication & Identity

**Auth Provider:**
- Cinatra platform (organization-level approval gate)
  - Implementation: BPMN `review_publish_bundle` user task with `cinatra:taskKind value="approval"` and `cinatra:approvalConfig level="organization" rejectionPolicy="needs_revision"` — approval is enforced by the Cinatra workflow engine, not by code in this repo

## Monitoring & Observability

**Error Tracking:**
- None detected in this repo

**Logs:**
- Workflow task status is surfaced in the operator dashboard via the `publish-status` portlet (`workflow-status` kind), consuming `projectId` from the dashboard scope

## CI/CD & Deployment

**Hosting:**
- Cinatra Marketplace (`registry.cinatra.ai`) — published on GitHub Release via `.github/workflows/release.yml`

**CI Pipeline:**
- GitHub Actions
  - `.github/workflows/ci.yml` — runs on push/PR to `main`; classifies repo as source-mirror (host-internal `@cinatra-ai/*` peers detected) and skips standalone install/typecheck/test; runs `npm pack --dry-run` for package shape validation; runs `extension-kind-gate.mjs` for BPMN workflow profile gate
  - `.github/workflows/release.yml` — calls the reusable `cinatra-ai/.github/.github/workflows/reusable-extension-release.yml@main` on GitHub Release publish; requires `CINATRA_MARKETPLACE_VENDOR_TOKEN` org secret

## Environment Configuration

**Required env vars:**
- None required at this package level
- `CINATRA_MARKETPLACE_VENDOR_TOKEN` — org-level GitHub Actions secret used by the release workflow (not configured in this repo directly)

**Secrets location:**
- No secrets stored in this repo; WordPress credentials are managed by the Cinatra platform's MCP connector layer identified via `wordpressInstanceId`

## Webhooks & Callbacks

**Incoming:**
- None — this is a static extension (BPMN + dashboard JSON + empty TS entry point)

**Outgoing:**
- Notification via BPMN `notify_publish_checkpoint_complete` send task: posts a message "Blog post {{postId}} published to {{wordpressInstanceId}}." on workflow completion — delivered by the Cinatra platform messaging layer, not by code in this repo

## Cinatra Platform Integrations (Internal)

**Agents referenced:**
- `@cinatra-ai/blog-wordpress-publish-agent` — invoked as a service task in `cinatra/workflow.bpmn` to create the WordPress draft
- Blog Pipeline agent (drafting + ideation) — referenced in README as a collaborating agent; not directly wired in the BPMN

**Cinatra object types consumed:**
- `@cinatra-ai/assets:blog-project` — project picker portlet in `cinatra/dashboard.json`
- `@cinatra-ai/assets:blog-idea` — ideas pane portlet
- `@cinatra-ai/assets:blog-post` — posts pane portlet

**Cinatra platform primitives used:**
- `blog_post_update` — ref-swap primitive for the draft editor portlet
- `blog_image_generate_start` — generation primitive for hero image portlet

---

*Integration audit: 2026-06-09*
