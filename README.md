# Blog Content Publish

A review-gated workflow + operator dashboard for shipping a blog post end-to-end. The extension composes the project → idea → post selection chain, an inline markdown editor with version history, a hero-image preview, and a publish launcher that drives a WordPress draft via an approval-gated BPMN workflow.

## Works with

- Blog Pipeline agent (drafting + ideation)
- Blog WordPress Publish agent (creates the draft from the post artifact)
- Cinatra Workflow Gantt (visualizes the publish DAG)
- WordPress MCP connector (instance picker + draft creation)

## Capabilities

- Pick a blog project, drill into ideas, then posts — one continuous selection chain in a single dashboard
- Edit the post body in place; every save mints a new artifact + swaps the post's `postArtifactId` ref (auditable via the version-history portlet)
- Preview the current hero image and see the auto-regen mode advertised by the post's `imageArtifactId`
- Launch the publish workflow with typed pickers for `projectId` / `postId` / `wordpressInstanceId` (credentials never reach the client)
- Watch the publish workflow's tasks + status update in the dashboard's `workflow-status` portlet
- Restart a publish with the same `{projectId, postId, wordpressInstanceId}` and get a structured idempotent-noop response instead of a duplicate WordPress draft
- Built on the workflow-extension surfaces doctrine and the Cinatra BPMN Profile (see [workflow-extension-surfaces](https://docs.cinatra.ai/references/platform/workflow-extension-surfaces.md) and [cinatra-bpmn-profile](https://docs.cinatra.ai/references/platform/cinatra-bpmn-profile.md))
