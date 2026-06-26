# Blog Content Publish

A review-gated workflow + operator dashboard for shipping a blog post end-to-end. The extension composes the project → idea → post selection chain, an inline markdown editor with version history, a hero-image preview, and a publish launcher that drives a WordPress draft via an approval-gated BPMN workflow.

**Purpose.** Installs a `blog-operator-dashboard` workspace in Cinatra covering the full blog-post lifecycle: project scoping, post editing, hero-image generation, and gated WordPress publication.

**Install.** Install from the Cinatra Marketplace. The dashboard appears after installation. WordPress credentials are managed through the WordPress MCP connector and never stored in this extension.

**Configuration.** The publish launcher prompts for `projectId`, `postId`, and `wordpressInstanceId` via typed object-pickers. An organisation-level approval step in `cinatra/workflow.bpmn` gates draft creation.

**Development.** Extension kind: `workflow`. The BPMN DAG is in `cinatra/workflow.bpmn`; portlet layout in `cinatra/dashboard.json`; `src/index.ts` is the package entry point with no runtime surface. Validate locally: `node extension-kind-gate.mjs --package-root .`. Host-internal peer dependencies are resolved by the Cinatra monorepo; standalone install is skipped in CI. See [workflow-extension-surfaces](https://docs.cinatra.ai/references/platform/workflow-extension-surfaces.md) and [Cinatra BPMN Profile](https://docs.cinatra.ai/references/platform/cinatra-bpmn-profile.md).

## Works with

- Blog Pipeline agent (drafting + ideation)
- Blog WordPress Publish agent (creates the draft from the post artifact)
- Cinatra Workflow Gantt (visualizes the publish DAG)
- WordPress MCP connector (instance picker + draft creation)

## Capabilities

- Pick a blog project, drill into ideas, then posts — one continuous selection chain in a single dashboard
- Edit the post body in place; every save mints a new artifact + swaps the post's `postArtifactId` ref (auditable via the version-history portlet)
- Preview the current hero image; the portlet reflects the auto-regen mode from the post's `imageArtifactId`
- Launch the publish workflow with typed pickers for `projectId` / `postId` / `wordpressInstanceId` (credentials never reach the client)
- Watch the workflow's tasks + status live in the `workflow-status` portlet
- Re-launch the workflow for the same post; the downstream agent handles duplicate-draft prevention
