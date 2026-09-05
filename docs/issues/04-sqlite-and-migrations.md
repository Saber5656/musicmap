# Title

SQLite connection, migration runner, and schema 001

## Summary

Implement the SQLite layer per ADR-006: better-sqlite3 connection factory with pragmas, a
forward-only SQL migration runner with pre-migration backup, and migration `001_init.sql`
containing the full DESIGN.md §5 schema.

## Context

All persistence flows through this layer. The schema in DESIGN.md §5 is normative — copy it
exactly; any deviation must be raised as a design question, not improvised.

## Scope

- `src/db/connection.ts`, `src/db/migrate.ts`, `src/db/migrations/001_init.sql`.
- Test helpers: `src/db/testing.ts` (open temp-file DB, run migrations).
- Add dependency `better-sqlite3` (+ `@types/better-sqlite3` dev).

## Detailed Requirements

1. `openDatabase(dbPath: string, opts?: { readonly?: boolean }): Database`
   - Pragmas on open: `journal_mode = WAL`, `foreign_keys = ON`, `busy_timeout = 5000`,
     `synchronous = NORMAL`.
   - On POSIX, `chmod 0600` the db file after creation (best effort).
   - Runs `PRAGMA quick_check` when the file pre-exists; on failure throw `ERR_DB_CORRUPT`
     with the message from SQLite (server startup handling is issue 07's).
2. Migration runner `migrate(db, migrationsDir, opts: { dbPath: string })`:
   - Migration files: `NNN_name.sql` (zero-padded ascending). Runner reads directory, sorts,
     compares against `schema_migrations` rows. Resolve `migrationsDir` via a single
     exported `migrationsDir()` helper (module-URL-relative) — packaging (issue 33) later
     points it at `dist/db/migrations`.
   - No hash tracking in v1: refuse to run ONLY when `MAX(applied.version)` >
     `MAX(on-disk version)` (`ERR_MIGRATION_FAILED`, details explain a downgrade). Edited or
     missing lower-numbered files are NOT detected in v1 (documented).
   - Backup before applying: only when `schema_migrations` already exists AND there are
     pending migrations — copy `opts.dbPath` → `<dbPath>.bak-<yyyymmddHHMMSS>` (DESIGN §14).
     Brand-new/empty DBs are never backed up.
   - Each migration runs inside a transaction; on error, rollback and throw
     `ERR_MIGRATION_FAILED` including file name and SQLite error.
   - Records `(version, name, applied_at)` into `schema_migrations` (created implicitly by the
     runner if absent, NOT by 001).
3. `001_init.sql`: the exact DDL from DESIGN.md §5 (all tables + indexes:
   `imports`, `artists`, `tracks`, `play_events`, `chapters`, `settings`,
   `artist_enrichment`, `genres`, `artist_genres`, `artwork`, `agg_monthly_artist`,
   `agg_meta`). `schema_migrations` is NOT in 001 — the runner owns it (per the §5 DDL
   comment).
4. `src/db/testing.ts`: `openTestDb(): { db, dir, close() }` using `mkdtemp` + real file (WAL
   needs a file; do not use `:memory:`), migrations applied.
5. better-sqlite3 must be listed under `dependencies` (runtime), and CI (issue 02) must pass
   with the native build on ubuntu + macos runners (prebuilds expected; if a runner compiles
   from source, that is acceptable as long as CI passes).

## Acceptance Criteria

- [ ] Fresh DB: `migrate` applies 001; `schema_migrations` has one row; all §5 tables/indexes
      exist (assert via `sqlite_master` in tests, comparing the full expected name list).
- [ ] Re-running `migrate` is a no-op (idempotent).
- [ ] A deliberately broken second migration (test fixture) rolls back atomically: no partial
      objects, `schema_migrations` unchanged, `.bak-` file exists.
- [ ] `foreign_keys` enforced: inserting a `play_events` row with unknown `track_id` throws.
- [ ] `ON DELETE CASCADE` verified: deleting an `imports` row removes its `play_events`.
- [ ] Corrupt file (test writes garbage bytes) → `ERR_DB_CORRUPT`.
- [ ] CHECK constraints verified for `imports.status`, `play_events.played_at_precision`,
      `chapters.title` length.
- [ ] T8 discipline: all runtime SQL in connection/migration metadata paths uses prepared
      statements; `db.exec` appears ONLY for trusted migration file contents from
      `migrationsDir()` (grep + review).

## Validation

`npm test` (suite `src/db/*.test.ts`) on macOS locally and ubuntu CI; `npm run lint`;
`npm run typecheck`.

## Dependencies

01-project-scaffold.

## Non-goals

Query/repository functions (13, 15, 16, and feature issues); agg rebuild logic (15); runtime
wiring of db path from config (07/14).

## Design References

DESIGN.md §5 (normative DDL), §14 (backup/corruption rows); ADR-006.
