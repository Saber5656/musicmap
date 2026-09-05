# Title

Config resolution, data directory bootstrap, and logging

## Summary

Implement configuration resolution (CLI flag > env > `config.json` > default), the per-OS data
directory bootstrap with restrictive permissions, and the pino logger with token redaction and
boot-time log rotation, per DESIGN.md §17 and §13.2 T10/T14.

## Context

Every subsystem (DB path, raw import storage, artwork cache, logs) resolves locations through
this module. It must be pure enough to test with temp dirs and injected env.

## Scope

- `src/config/index.ts` (new directory; server/cli/importers may import it, core may not).
- `src/logger.ts`.
- Unit tests for both.

## Detailed Requirements

1. Types & schema (zod):
   ```ts
   type ConfigSource = 'cli' | 'env' | 'file' | 'default';
   interface ResolvedConfig {
     port: number;            // default 4949, range 1024-65535
     dataDir: string;         // absolute path
     openBrowser: boolean;    // default true
     logLevel: 'debug'|'info'|'warn'|'error';  // default 'info'
   }
   interface ResolveResult {
     config: ResolvedConfig;
     sources: Record<keyof ResolvedConfig, ConfigSource>;  // issue 07 needs port explicitness
     warnings: string[];      // caller logs these (resolveConfig itself never logs)
   }
   ```
   Explicitness rule (consumed by issues 06/07 port-fallback): `port` counts as explicit
   when `sources.port` is `'cli'` or `'env'`; a `config.json` port is NOT explicit
   (documented).
2. `resolveConfig(cli: Partial<ResolvedConfig>, env = process.env): ResolveResult`:
   - Precedence per DESIGN §17.1. Env vars: `MUSICMAP_PORT`, `MUSICMAP_DATA_DIR`,
     `MUSICMAP_LOG_LEVEL` (no env var for `openBrowser`: its sources are cli > file >
     default only).
   - `config.json` location: `env-paths('musicmap', { suffix: '' }).config + '/config.json'`;
     file optional. Failure semantics (both tested): (a) syntax error or type-invalid values
     → warning + the ENTIRE file is ignored; (b) valid file with unknown keys → warning per
     unknown key, known keys still apply. Never crash on bad config.
   - Default `dataDir`: `env-paths('musicmap', { suffix: '' }).data`.
3. `ensureDataDirs(dataDir: string): void` creates (recursive) `dataDir`, `imports/`,
   `artwork/`, `logs/`; on POSIX sets mode `0o700` on `dataDir` (best effort — ignore
   `EPERM`/Windows). Symlink policy (T10): `lstatSync` each of the four paths that already
   exist — a symlink or a non-directory at any of them → throw `ERR_DATA_DIR_INVALID`
   (POSIX symlink tests required).
4. Logger (`src/logger.ts`):
   - `createLogger(cfg): pino.Logger` writing pretty to stderr when `process.stderr.isTTY`
     and `NODE_ENV !== 'production'`, plus JSON lines to `<dataDir>/logs/musicmap.log`.
   - Redaction: `redact: { paths: ['req.headers.authorization', 'authorization', 'token'],
     censor: '[redacted]' }`.
   - Boot rotation: before opening the file, if `musicmap.log` > 10 MiB, shift
     `musicmap.log.3→.4(delete oldest) ... .1→.2, log→.1` keeping at most 5 files total.
5. Export a small `paths(dataDir)` helper: `{ db: <dataDir>/musicmap.db, importsDir,
   artworkDir, logsDir, lockFile: <dataDir>/lock }` — single source for all path building
   (later issues must use it; no ad-hoc `path.join(dataDir, ...)` elsewhere).
6. No dependency additions beyond `env-paths`, `pino`, `zod` (already allowlisted).

## Acceptance Criteria

- [ ] Precedence proven by tests: flag beats env beats file beats default, per field;
      `sources` reports the winning layer per field (incl. `openBrowser` having no env
      source).
- [ ] Syntax-error/type-invalid `config.json` → whole file ignored + warning in
      `warnings[]`; unknown-keys file → per-key warnings, known keys applied; process never
      crashes; `resolveConfig` itself writes nothing to stdout/stderr (warnings returned,
      not printed).
- [ ] `ensureDataDirs` creates the tree; POSIX mode of `dataDir` is `0700` on macOS/Linux test.
- [ ] Existing-file-at-dataDir AND symlink-at-any-managed-path cases throw
      `ERR_DATA_DIR_INVALID` (symlink cases POSIX-only tests).
- [ ] Log file receives JSON lines; `authorization` value never appears in any log line
      (test by logging a fake req object).
- [ ] Rotation test: pre-create an 11 MiB log, boot logger, assert shift to `.1` and new empty
      log.
- [ ] All tests use temp dirs (`fs.mkdtempSync`) — nothing touches the real user dirs.

## Validation

`npm test` (new suites `src/config/*.test.ts`, `src/logger.test.ts`); `npm run lint`;
`npm run typecheck`.

## Dependencies

01-project-scaffold.

## Non-goals

Settings stored in the DB (`settings` table — issue 24); CLI flag parsing itself (07); reading
config inside the server (06/07 wire it).

## Design References

DESIGN.md §17.1, §17 (data dir layout), §13.2 T10/T14, §14 (port-fallback lives in issue
06's `listenLoopback` helper; serve-time UX in issue 07 — both consume `sources.port`).
