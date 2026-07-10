# ADR-006: SQLite with raw SQL and versioned migrations (no ORM)

- Status: Accepted (2026-07-10)
- Deciders: Fable (design)

## Context

The data layer is analytics-heavy (window functions, group-bys over ~10^6 rows) and will be
implemented and modified by lower-capability agents. ORMs abstract exactly the part that matters
here (SQL shape and indexes) while adding schema-drift and version-churn risk (e.g. pre-1.0
Drizzle APIs).

## Decision

- **better-sqlite3** (synchronous, prepared statements) as the only DB driver; WAL mode;
  `foreign_keys = ON`.
- Schema managed by **numbered SQL migration files** (`src/db/migrations/NNN_name.sql`) applied
  in order inside a transaction by a ~50-line migration runner; applied versions recorded in
  `schema_migrations`. Forward-only (no down migrations).
- All queries are **hand-written SQL with bound parameters**, wrapped in typed repository
  functions (`src/db/queries/*.ts`) returning validated TS types. String interpolation into SQL
  is forbidden (lint rule + review checklist).
- Heavy read paths hit **pre-aggregation tables** (`agg_monthly_artist`, `agg_monthly_genre`)
  rebuilt transactionally after imports/deletions/enrichment changes.

## Consequences

- The exact SQL is visible in issues and reviewable; performance work = SQL + index work.
- Migration discipline is required for every schema change (checklist in CONTRIBUTING).
- No query builder ergonomics; mitigated by keeping all SQL in the queries layer with tests.

## Alternatives considered

- **Drizzle/Prisma**: type-safe builders, but added deps, codegen, and churn for a schema of
  ~12 tables; rejected.
- **DuckDB**: superb analytics, but WASM/native packaging complexity and single-writer app needs
  transactional app-state too; SQLite covers both adequately at our scale; rejected.
- **node:sqlite (built-in)**: still maturing across the Node 24 line and lacks better-sqlite3's
  performance track record; revisit when stable (noted as v2 simplification).
