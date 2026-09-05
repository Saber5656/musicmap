# Title

CLI: `serve` and `status` commands

## Summary

Implement the `musicmap` CLI entry (commander) with `serve` (boot server, print tokenized URL,
open browser, graceful shutdown, single-writer lock) and `status` (data dir, DB presence/size,
row counts), per DESIGN.md §3.3, §17.1, §14.

## Context

The CLI is the only user entry point. `musicmap import` is added later by issue 14 — structure
the command registry so a new command file can be added without touching existing ones.

## Scope

- `src/cli/index.ts` (program setup), `src/cli/commands/serve.ts`, `src/cli/commands/status.ts`.
- Add dependency `open` (allowlisted). Uses `commander` (installed in 01).

## Detailed Requirements

1. Program: `musicmap` with `--version` (from package.json via `createRequire` or build-time
   import of `package.json` with `resolveJsonModule`), `--help`. Global options none; each
   command declares its own.
2. `musicmap serve [--port <n>] [--data-dir <path>] [--no-open]`:
   - Resolve config via issue 03 (`{ port, dataDir, openBrowser }` from flags).
   - `ensureDataDirs`, then acquire the single-instance write lock BEFORE any writable DB
     access (DESIGN §14 order): atomic exclusive create (`flag 'wx'`) of `<dataDir>/lock`
     containing the PID. Lock held by a live PID → start in **read-only warn mode**: open
     the DB read-only, skip migrate and recovery, set `readOnly: true` in `buildApp` opts
     (issue 14 rejects imports with 409 when set), print a warning. Dead PID → take over
     the stale lock (debug log only, no user-facing warning). Remove lock on clean shutdown.
   - Lock acquired → open DB writable + migrate (issue 04, passing `dbPath`).
   - Interrupted-import recovery (owned HERE, direct prepared-statement SQL — not an issue
     09 API): on boot, set any `imports` rows in `uploaded`/`parsing` to `failed` with
     `error_json = { code: 'ERR_INTERRUPTED' }` (skipped in read-only mode).
   - Decoration wiring after `buildApp`: register `mmDb` (the open Database) and
     `mmDataDir` per issue 06's reserved decoration names.
   - Build app (issue 06) with a fresh token; bind via issue 06's
     `listenLoopback(app, port, { explicit })` where `explicit` derives from issue 03's
     `sources.port` (`'cli' | 'env'`); print exactly one line to stdout:
     `musicmap listening on http://127.0.0.1:<boundPort>/#token=<token>`; when
     `openBrowser`, call `open(url)` (never in `NODE_ENV === 'test'`).
   - Graceful shutdown on SIGINT/SIGTERM: stop accepting (fastify `close()`), close DB,
     remove lock, exit 0 within 5 s (force-exit 1 after timeout).
   - DB corruption (`ERR_DB_CORRUPT` from issue 04): print recovery guidance (path of DB,
     suggestion to restore `musicmap.db.bak-*` or move the file away) and exit 1.
3. `musicmap status [--data-dir <path>]`:
   - Prints (plain text, stable `key: value` lines): `data_dir`, `database`
     (`present`/`missing`), `database_size_bytes` (0 when missing), `schema`, counts
     (imports by status, play_events, artists, tracks, chapters), enrichment enabled +
     matched/pending counts (0s when tables empty), `log_file`.
   - Exits 0 even when DB absent ("no database yet — run `musicmap serve`").
   - Never modifies anything: open read-only WITHOUT the issue-04 write pragmas/migrate.
     Schema inspection is defensive: `schema_migrations` table absent → `schema: none
     (outdated)`; `MAX(applied) < MAX(bundled)` → `schema: v<N> (outdated — will migrate on
     serve)`; `MAX(applied) > MAX(bundled)` → `schema: v<N> (unsupported — newer than this
     binary)`; equal → `schema: v<N> (current)`.
4. Exit codes: 0 success; 1 fatal (port explicit+busy, corrupt DB, invalid data dir); 2 usage
   error (commander default behavior acceptable).
5. All user-facing CLI strings in English (CLI is not i18n'd in v1 — only the web UI is).

## Acceptance Criteria

- [ ] `musicmap serve` on a fresh dir: creates tree, migrates, prints
      `musicmap listening on http://127.0.0.1:4949/#token=...`, serves `/api/health`.
- [ ] Second concurrent `serve` on same data dir: picks port 4950, warns read-only mode, its
      `mmReadOnly` decoration is true (integration test via injected app state or log).
- [ ] Stale lock (dead PID) is replaced silently.
- [ ] SIGINT → lock removed, exit 0. Kill -9 → next boot replaces stale lock and marks
      in-flight imports `failed`/`ERR_INTERRUPTED` (test by seeding a `parsing` row).
- [ ] `--port 4949` while busy → exit 1 with message (no fallback); without flag → fallback
      port chosen.
- [ ] `status` output contains every listed line on both empty and seeded DBs; makes no writes
      (mtime of DB unchanged).
- [ ] `--no-open` skips `open`; test stubs `open` and asserts not called.

## Validation

`npm test` (CLI tests spawn via `execa`-less `child_process` with `MUSICMAP_DATA_DIR` temp
dirs, or call command handlers directly with injected deps — prefer direct handler calls +
one real spawn smoke test), lint, typecheck.

## Dependencies

03-config-and-paths, 06-server-security-bootstrap (04 transitively).

## Non-goals

`import` command (14), packaging/bin smoke in CI (33), i18n of CLI output, daemonization.

## Design References

DESIGN.md §3.3, §17.1, §14 (port busy / two instances / interrupted / corruption rows),
ADR-004 (token URL).
