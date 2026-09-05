# ADR-001: Local-first single-process architecture

- Status: Accepted (2026-07-10)
- Deciders: product owner (Saber5656), Fable (design)

## Context

musicmap visualizes personal music playback history — sensitive personal data. The frozen
requirements mandate a locally-run web app whose history data never leaves the machine, released
as OSS for self-use. Implementation will be executed by lower-capability agents, so the
architecture must minimize moving parts and cross-component contracts.

## Decision

A **single npm package** running a **single Node.js process**:

- Node.js >= 24 (current Active LTS), TypeScript (strict), ESM.
- **Fastify 5** HTTP server bound to `127.0.0.1` only, serving both the JSON API (`/api/*`) and
  the built SPA static assets.
- **SQLite** (better-sqlite3, WAL mode) as the only persistence, in a per-OS user data directory.
- **React 19 + Vite** SPA for the UI; d3 for visualizations.
- A thin **CLI** (`musicmap serve|import|status`) as the process entry point.
- No daemon, no external services, no database server, no Docker requirement.

## Consequences

- Zero-config startup; the entire state is one directory (easy backup/delete).
- Synchronous better-sqlite3 keeps the data layer simple and fast for read-heavy analytics, at
  the cost of blocking the event loop on long statements — mitigated by pre-aggregation tables
  and streaming imports that batch writes in transactions.
- Single process means the enrichment job and imports run in-process; long tasks must be
  incremental and resumable (see DESIGN.md §12).
- Serving SPA and API from one origin removes CORS entirely and simplifies the security model
  (ADR-004).

## Alternatives considered

- **CLI + static HTML report**: simplest, but interactivity (chapters editing, genre map
  exploration) requires a live API; rejected.
- **Tauri/Electron desktop app**: better install UX but heavy build/signing/update burden for a
  v1 OSS project; rejected (revisit v2).
- **Hosted multi-user service**: violates the privacy requirement; rejected.
- **Monorepo with separate packages (core/server/web)**: better layering but more workspace
  tooling for implementation agents to get wrong; single package with enforced directory
  boundaries chosen instead.
