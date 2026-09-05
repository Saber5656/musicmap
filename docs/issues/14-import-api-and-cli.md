# Title

Import HTTP endpoints and `musicmap import` CLI command

## Summary

Wire the import framework to users: multipart upload endpoint with size limits, import
list/detail/progress/delete endpoints, duplicate handling, read-only-mode rejection, and the
headless `musicmap import <file> --source <s>` command — per DESIGN.md §6.1-6.2, §6.6, §9,
§14.

## Context

This issue integrates issues 06-13 into the first end-to-end capability: files in → events in
DB. UI comes later (18); this ships the API contract and CLI path with full tests.

## Scope

- `src/server/routes/imports.ts`; registration in `buildApp`.
- `src/cli/commands/import.ts`.
- Add dependency `@fastify/multipart`.

## Detailed Requirements

1. `POST /api/imports` (multipart):
   - Fields: `source` (must be one of the three enum values; else 400 `ERR_VALIDATION`) and
     `file` (exactly one; missing → 400). Limits: `fileSize: 4 GiB`, `files: 1`,
     `fields: 5` (normative per DESIGN §6.2); exceeding size → 413 `ERR_UPLOAD_TOO_LARGE`
     (map fastify's limit error).
   - Read-only mode (`mmReadOnly`, issue 07) → 409 `ERR_CONFLICT`
     (`details.reason = 'read_only'`).
   - Exact sequence (normative — `saveUpload` needs the id): (1) generate import id
     (UUID v4); (2) `saveUpload(dataDir, id, filename, stream)`; (3)
     `findDuplicate(source, sha256)` → on hit, delete the just-saved raw dir, do NOT insert
     a row, return 409 `ERR_DUPLICATE_UPLOAD` with `details.existingImportId`; (4)
     `createImport({ id, ..., status: 'uploaded' })`; (5) `ImportService.enqueue(id)`;
     (6) reply `202 { id }`.
2. `GET /api/imports` → `{ imports: [ { id, source, status, originalFilename, createdAt,
   finishedAt, totals? (from report: eventsImported, duplicates, earliest/latest) , errorCode? } ] }`
   newest-first (no raw report/error dumps in the list).
3. `GET /api/imports/:id` → full record: report_json parsed, error_json parsed, plus
   `progress` (from `ImportService.progress`) while `parsing`. Unknown id → 404.
4. `DELETE /api/imports/:id` — status matrix (per DESIGN §6.1): `completed`/`failed` →
   `deleteImportFully` → 204; `uploaded` (queued, not yet started) →
   `ImportService.cancelQueued(id)` + `deleteImportFully` → 204; `parsing` → 409
   `ERR_CONFLICT` (`details.reason = 'parsing'`); read-only mode → 409
   (`details.reason = 'read_only'`).
5. Zod schemas for params/query/replies co-located in the route file; all handlers typed.
6. Parser registry: `const parsers = [spotifyParser, appleParser, youtubeParser]` wired in
   server bootstrap AND reused by CLI (single module `src/importers/registry.ts`).
7. CLI `musicmap import <file> --source <s> [--data-dir <path>]`:
   - Accepted `--source` values (exhaustive): `spotify`, `apple`, `apple_music`, `ytm`,
     `youtube_music` — aliases resolve to the canonical enum before reaching the framework.
   - No server required: acquires the issue-07 write lock (`<dataDir>/lock` PID file, same
     helper — atomic create, stale-PID takeover); a LIVE PID holding it → exit 1 with
     "stop the server or use the web UI"; then opens the DB writable + migrates.
   - Copies the file via `saveUpload` (sha256 duplicate check applies — exit 1 with existing
     import id), runs the import synchronously with a progress line (`\r`-updated
     `files X/Y · events N · MB read/total`), prints the report summary table (totals +
     skipped breakdown) and exits 0 on `completed`, 1 on `failed` (printing error code +
     message).
   - `--json` flag: machine-readable single-line JSON result instead of the table.
8. Failure-mode rows owned here (DESIGN §14): wrong-source message surfaces
   `ERR_NO_RECOGNIZED_FILES` details (`foundEntries`, `didYouMean`, `hint` — issue 09
   shapes) in both the API error envelope and CLI output (CLI prints up to 10 found
   entries).

## Acceptance Criteria

- [ ] fastify-inject: upload of the issue-10 mini ZIP fixture → 202; polling detail reaches
      `completed` with expected totals; events queryable.
- [ ] Missing/invalid `source`, missing file, two files, oversize (test with lowered
      per-test limit override) → 400/400/400/413 with correct codes.
- [ ] Duplicate sha256+source → 409 with `existingImportId`; raw dir of the rejected upload
      removed (fs asserted).
- [ ] Delete status matrix: during parsing → 409; queued (`uploaded`) → 204 with raw dir
      removed and job dequeued; after completion → 204 + events gone + raw dir gone.
- [ ] Read-only mode: POST and DELETE → 409 `read_only`; GETs still work.
- [ ] Security matrix on POST and DELETE (T1/T2/T11): missing token → 401; wrong token →
      401; token as query param only → 401; valid token + hostile Host → 403; valid
      token + valid Host → success; no token value appears in any response body or captured
      log line.
- [ ] CLI: happy path exit 0 with summary; wrong source shows didYouMean hint; duplicate
      exits 1 with existing id; `--json` output parses and matches schema.
- [ ] Lock contention: CLI import while a fake lock (live PID) exists → exit 1 message.

## Validation

`npm test` (suites `src/server/routes/imports.test.ts`, `src/cli/import.test.ts`), lint,
typecheck. End-to-end: run `musicmap import fixtures/exports/spotify/<mini>.zip --source
spotify` locally and paste the summary in the PR.

## Dependencies

06, 07, 09, 10, 11, 12, 13.

## Non-goals

Import UI (18), SSE/websocket progress (polling only), resumable uploads, agg rebuild content
(15 — but the `onCompleted` hook must already be invoked; a no-op placeholder is wired here
and replaced in 15).

## Design References

DESIGN.md §6.1, §6.2, §6.6, §6.7, §9 (endpoints/envelope), §14 (wrong source, read-only);
ADR-004 (auth applies).
