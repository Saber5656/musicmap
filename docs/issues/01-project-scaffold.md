# Title

Project scaffold: TypeScript/ESM single package with lint, test, and build skeleton

## Summary

Create the complete repository skeleton per DESIGN.md §3.2/§3.3/§4: one npm package (ESM,
TypeScript strict, Node >= 24) containing server/CLI code built with `tsc` and a React SPA
built with Vite, plus ESLint (flat), Prettier, Vitest, and npm scripts. No product features —
only a compiling, testable, lintable skeleton with a "hello" page and passing sample test.

## Context

musicmap is a local-first web app (ADR-001): one Node process serves both a JSON API and the
built SPA. All later issues assume this exact layout, the dependency allowlist, and the
layering rules, so this issue defines them mechanically.

## Scope

- `package.json`, `tsconfig.json`, `tsconfig.build.json`, `vite.config.ts`,
  `vitest.config.ts`, `eslint.config.js`, `.prettierrc.json`, `.gitignore`, `.editorconfig`,
  `LICENSE`, stub `README.md`.
- Directory skeleton under `src/` per DESIGN.md §3.2 with placeholder modules that compile.
- npm scripts: `dev`, `build`, `start`, `test`, `lint`, `format`, `typecheck`.

## Detailed Requirements

1. `package.json`:
   - `name: "musicmap"`, `version: "0.1.0"`, `"type": "module"`, `license: "MIT"`,
     `engines: { "node": ">=24" }`, `bin: { "musicmap": "dist/cli/index.js" }`,
     `files: ["dist", "README.md", "LICENSE"]`.
   - Dependencies ONLY from DESIGN.md §4 allowlist; this issue installs: `react`, `react-dom`,
     `react-router-dom`, `i18next`, `react-i18next`, `commander`, `fastify`,
     `@fastify/static`, `pino`, `zod`, `env-paths`. devDependencies: `typescript`, `vite`,
     `@vitejs/plugin-react`, `vitest`, `eslint` + `typescript-eslint` + `eslint-plugin-react-hooks`,
     `prettier`, `tsx`, `concurrently`, `@types/node`, `@types/react`, `@types/react-dom`,
     `pino-pretty`. Rule: `@types/*` packages for allowlisted deps are always permitted as
     devDependencies without escalation. (Other allowlisted deps are added by the issues that
     use them.)
   - Scripts:
     - `"dev"`: `concurrently -k "tsx watch src/server/index.ts" "vite"`
     - `"build"`: `tsc -p tsconfig.build.json && vite build`
     - `"start"`: `node dist/cli/index.js serve`
     - `"test"`: `vitest run`; `"lint"`: `eslint .`; `"format"`: `prettier --write .`;
       `"typecheck"`: `tsc -p tsconfig.json --noEmit`
2. TypeScript — explicit three-config split (web code is bundler-resolved, node code is
   NodeNext; one config cannot serve both):
   - `tsconfig.base.json`: shared flags only — `strict: true`, `target: "ES2023"`,
     `skipLibCheck: true`, `noUncheckedIndexedAccess: true`, `verbatimModuleSyntax: true`.
   - `tsconfig.build.json` (extends base): `module`/`moduleResolution` `"NodeNext"`,
     `outDir: "dist"`, `rootDir: "src"`, `include: ["src/cli", "src/server", "src/core",
     "src/importers", "src/db", "src/enrichment"]` (never `src/web`).
   - `tsconfig.web.json` (extends base): `module: "ESNext"`,
     `moduleResolution: "Bundler"`, `jsx: "react-jsx"`, `noEmit: true`,
     `include: ["src/web", "src/core/types.ts", "src/core/errors.ts"]` (the shared-types
     surface web may import per the layering rules).
   - Root `tsconfig.json`: solution style — `{ "files": [], "references": [
     { "path": "./tsconfig.build.json" }, { "path": "./tsconfig.web.json" } ] }` (editor
     support; add `composite: true` to the two project configs as required by references).
   - `"typecheck"` script: `tsc -p tsconfig.build.json --noEmit && tsc -p tsconfig.web.json`.
3. Vite: `root: "src/web"`, `plugins: [react()]`,
   `build: { outDir: "../../dist/web", emptyOutDir: true }`,
   `server: { port: 5173, proxy: { "/api": "http://127.0.0.1:4949" } }`.
4. Directory skeleton (each with a minimal compiling placeholder, no logic):
   `src/server/app.ts` exports `buildApp()` returning a Fastify instance answering
   `GET /api/health` with `{ ok: true, version }`; `src/server/index.ts` exports
   `startServer()` that calls `buildApp()` and listens on `127.0.0.1:4949` — a
   scaffold-only bootstrap that issue 06 replaces with the hardened startup (the loopback
   host is already mandatory here); `src/cli/index.ts` (shebang `#!/usr/bin/env node`):
   no args → prints version, exits 0; `serve` arg → calls `startServer()` (so `npm start`
   and `npm run dev` work end-to-end from day one); `src/core/types.ts`,
   `src/core/errors.ts`, `src/web/index.html`, `src/web/main.tsx` (renders "musicmap"
   heading), `src/web/styles/tokens.css` (empty token stubs).
   `index.html` must contain **no inline scripts or styles** (DESIGN §13.3) — only
   `<script type="module" src="/main.tsx">` and a `<link>` to css.
5. ESLint flat config:
   - typescript-eslint recommended-type-checked on `src/**`, react-hooks on `src/web/**`.
   - Layering rules via `no-restricted-imports` (DESIGN §3.2): `src/core/**` may not import
     from `src/server|db|importers|enrichment|web`; `src/web/**` may not import from
     `src/server|db|importers|enrichment|cli` (only `src/core/types` and web-local modules).
   - Forbid `dangerouslySetInnerHTML` (`no-restricted-syntax` on the JSX attribute) — DESIGN
     §13.2 T7.
6. `.gitignore`: `node_modules/`, `dist/`, `*.log`, `.DS_Store`, `fixtures/generated/`,
   `test-results/`, `playwright-report/`.
7. `LICENSE`: MIT, copyright `2026 Saber5656`.
8. One sample vitest test (`src/core/types.test.ts` or similar) asserting a trivial truth to
   prove the pipeline.
9. Commit the `package-lock.json`.

## Acceptance Criteria

- [ ] `npm ci && npm run build` succeeds on Node 24; produces `dist/cli/index.js`,
      `dist/server/`, `dist/web/index.html`.
- [ ] `npm test`, `npm run lint`, `npm run typecheck` (both projects) all pass.
- [ ] `node dist/cli/index.js` prints the package version and exits 0; `npm start` serves
      `/api/health` and the built hello page on `http://127.0.0.1:4949`.
- [ ] `npm run dev` serves the hello page on `http://127.0.0.1:5173` with `/api` proxied to
      the running scaffold server on 4949.
- [ ] Layering lint rule demonstrably fails when e.g. `src/core/x.ts` imports from `src/db`
      (prove with a temporary file in the PR description or a lint unit test, then remove).
- [ ] No dependencies outside DESIGN.md §4; lockfile committed; `engines.node >= 24`.
- [ ] `dist/web/index.html` contains no inline `<script>`/`<style>`.

## Validation

Run: `npm ci`, `npm run build`, `npm test`, `npm run lint`, `npm run typecheck`,
`node dist/cli/index.js`. Paste outputs in the PR. CI does not exist yet (issue 02).

## Dependencies

None (first issue).

## Non-goals

CI (02), config/data dir (03), real server security (06), real CLI commands (07), any product
feature. Do not add `better-sqlite3` or parser deps yet.

## Design References

DESIGN.md §3.2 (layout), §3.3 (modes/scripts), §4 (stack & allowlist), §13.3 (no inline
scripts); ADR-001, ADR-006 (context only).
