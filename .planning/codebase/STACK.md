# Technology Stack

**Analysis Date:** 2026-06-09

## Languages

**Primary:**
- TypeScript 5.x - All source code under `src/` (compiled to ES2023, JSX via react-jsx)
- XML (BPMN) - Workflow definition at `cinatra/workflow.bpmn`
- JSON - Dashboard portlet composition at `cinatra/dashboard.json`

**Secondary:**
- JavaScript (ESM) - CI gate utility at `extension-kind-gate.mjs` (Node builtins only, zero dependencies)

## Runtime

**Environment:**
- Node.js 24 (specified in CI via `actions/setup-node@v4` with `node-version: "24"`)

**Package Manager:**
- pnpm (via corepack, `corepack enable` in CI)
- Lockfile: Not present (CI uses `--no-frozen-lockfile`; this is a source-mirror repo)

## Frameworks

**Core:**
- Cinatra Platform (via `@cinatra-ai/*` optional peer dependencies) - workflow + dashboard extension surfaces
- React (JSX transform configured via `"jsx": "react-jsx"` in `tsconfig.json`) - UI portlet rendering

**Testing:**
- Not applicable — no test files present; tests run in the Cinatra monorepo context

**Build/Dev:**
- TypeScript compiler (`tsc`) - configured via `tsconfig.json`, targets `ES2023`, outputs to `dist/`
- No bundler configured — `moduleResolution: "bundler"` in tsconfig delegates to the monorepo build chain

## Key Dependencies

**Critical:**
- `@cinatra-ai/*` packages (optional peerDependencies, not in package.json devDeps/deps) - host-internal packages provided by the Cinatra monorepo at install time; never published to a public registry

**Infrastructure:**
- No runtime npm dependencies declared in `package.json` — this is a content/config extension with no standalone runtime code (`src/index.ts` exports nothing; it is `export {}`)

## Configuration

**Environment:**
- No `.env` files detected
- No runtime environment variables required by this package directly; runtime credentials (WordPress credentials, etc.) are managed by the Cinatra platform at workflow execution time and never reach the client

**Build:**
- `tsconfig.json` - standalone strict TypeScript config, targets ES2023, ESNext modules, bundler resolution, outputs declarations + source maps to `dist/`
- `.npmrc` - sets `auto-install-peers=false`

## Platform Requirements

**Development:**
- Node.js 24+, pnpm via corepack
- Must be developed/typechecked inside the Cinatra monorepo workspace (host-internal `@cinatra-ai/*` peers resolve only there)

**Production:**
- Deployed as a Cinatra Marketplace extension via the `release.yml` GitHub Actions workflow
- Published through the Cinatra marketplace MCP proxy (extension-submit-for-review → marketplace approval saga → `registry.cinatra.ai`)
- Not standalone-installable from any public npm registry

---

*Stack analysis: 2026-06-09*
