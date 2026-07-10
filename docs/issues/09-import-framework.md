# Title

Import pipeline framework and lifecycle state machine

## Summary

Implement the source-agnostic import machinery per DESIGN.md §6: import record lifecycle
(`uploaded → parsing → completed|failed`), raw-file storage with SHA-256 duplicate detection,
single-worker orchestration with progress, batched event persistence with the §6.5 idempotent
insert policy, ImportReport accumulation, per-import failure cleanup, and import deletion with
cascade + hooks for identity GC and agg rebuild.

## Context

Parsers (10-12) are pure per-file streams; identity resolution (13) and agg rebuild (15) plug
in via injected interfaces. This framework owns ALL persistence and state transitions so those
pieces stay independently testable. HTTP/CLI surfaces (14) call into this.

## Scope

- `src/importers/framework.ts` (orchestrator), `src/importers/store.ts` (raw file store),
  `src/db/queries/imports.ts`, `src/db/queries/events.ts`.
- Unit/integration tests with a fake parser and temp DB.

## Detailed Requirements

1. Raw file store (`store.ts`):
   - `saveUpload(dataDir, importId, filename, stream): Promise<{ rawPath, absolutePath,
     sha256, bytes }>` (`rawPath` is the data-dir-relative `imports/<id>/original.<ext>`
     stored in `imports.raw_path`; `absolutePath` for immediate reads) —
     streams to `<dataDir>/imports/<importId>/original<ext>` (ext from a sanitized whitelist:
     `.zip`, `.json`, `.csv`; anything else → `.bin`; NEVER derived from user path segments),
     computing sha256 while writing. `ENOSPC` → `AppError('ERR_DISK_FULL')` and partial file
     unlinked.
   - `deleteRawFiles(dataDir, importId)` removes the import's directory (rm -rf equivalent via
     `fs.rm({ recursive: true, force: true })` on the exact known path only).
2. Import records (`queries/imports.ts`): `createImport`, `findDuplicate(source, sha256)`
   (statuses `uploaded|parsing|completed`), `setStatus`, `setReport`, `setError`,
   `listImports` (newest first), `getImport`, `deleteImport` (row delete; events cascade).
   (Interrupted-import boot recovery is owned by issue 07 as direct SQL — not part of this
   API.) All queries here and in `queries/events.ts` use prepared statements with bound
   parameters — no string-built SQL (T8).
3. Orchestrator (`framework.ts`):
   ```ts
   interface IdentityResolver {             // implemented by issue 13; full contract HERE
     resolve(e: CanonicalPlayEvent): { artistId: number; trackId: number;
                                       artistCreated: boolean; trackCreated: boolean };
     // created flags feed ImportReport.artistsCreated / tracksCreated (§6.6)
     gc(): { tracksDeleted: number; artistsDeleted: number };
     resetCache(): void;                    // framework calls this at import start
   }
   interface ImportDeps { db; dataDir; parsers: ImportSourceParser[];
     identity: IdentityResolver;
     onCompleted?: (importId) => void;      // issue 15 hooks agg rebuild
     log; }
   class ImportService {
     enqueue(importId): void;               // FIFO, single concurrent job
     progress(importId): ImportProgress | undefined;
     cancelQueued(importId): boolean;       // only while 'uploaded'
   }
   interface ImportProgress { phase: 'reading'|'finalizing'; bytesRead: number;
     bytesTotal: number; filesDone: number; filesTotal?: number; eventsImported: number; }
   ```
4. Job execution for import `I` of source `S` (parser selected by canonical enum only:
   `parsers.find(p => p.source === S)` — CLI aliases are resolved before this layer):
   - Transition `uploaded → parsing`; call `identity.resetCache()`.
   - Resolve input files: if raw file is a zip → `iterateZipEntries` (issue 08 — an
     implementation dependency of this issue), filter by the parser's
     `matches(entryName, peek)` where peek = first 4 KiB (buffer the stream head);
     if plain file → treat as single candidate (still `matches`-checked with its filename).
   - After matching, when the parser implements `selectFiles(matchedNames)`, apply it: parse
     only the returned `use` list and merge its `warnings` into the report (issue 11 uses
     this to suppress Daily Tracks when Play Activity is present).
   - Zero matched files → fail `ERR_NO_RECOGNIZED_FILES`; `details` include
     `foundEntries` (up to 50 entry names), `didYouMean: <otherSource>` when another
     registered parser matches ≥1 entry, AND `hint` — the first non-undefined result of the
     selected parser's `hintFor(entryName)` across found entries (issue 12's HTML-format
     hint).
   - For each matched file: run `parser.parse(stream, ctx)`; consume `ParsedRow`s:
     - `event` → resolve identity (in-memory cached; issue 13) → buffer.
     - `skip`/`warning` → count into per-file report (store ≤ 1,000 warning detail strings
       total; counters always exact).
   - Flush buffer every 5,000 events in ONE transaction: identity upserts + event inserts
     with the §6.5 conflict policy (`INSERT ... ON CONFLICT(event_hash) DO NOTHING` /
     day-precision `DO UPDATE` MAX rule). Duplicate counting semantics (normative): EVERY
     `event_hash` conflict increments `duplicates`, regardless of whether the day-precision
     MAX update changed values; there is no separate "updated" counter in v1.
   - Progress updated at flush boundaries from byte counters (`ctx.onBytes`).
   - Completion: write report (§6.6 shape incl. totals + earliest/latest played_at via one
     query), `parsing → completed`, then `onCompleted` hook (agg rebuild — issue 15 wires it;
     framework must tolerate hook absence).
   - Failure (any `AppError` or unexpected error): `DELETE FROM play_events WHERE import_id =
     I` in a transaction, set `failed` + `error_json` (`ERR_INTERNAL` for unexpected; log full
     error), keep raw file for diagnosis.
   - Parser row-level errors never fail the job (§6.7): parsers emit `skip:invalid` rows; only
     thrown errors are fatal.
5. Deletion flow `deleteImportFully(importId)`: reject while `parsing` (409 semantics — caller
   maps); transaction: delete import row (events cascade) → `identity.gc()` → commit → delete
   raw files → `onCompleted`-equivalent rebuild hook. Must be safe when raw files already
   missing.
6. Concurrency: a module-level FIFO ensures at most one running job; `enqueue` during a run
   queues; queued imports remain `uploaded`. Read-only mode (issue 07) is enforced by the API
   layer (14), not here.

## Acceptance Criteria

- [ ] Happy path with a fake parser (3 files, 12k events): batches of 5k, correct report
      totals, `completed`, progress monotonically increasing to 100%; `resetCache()` called
      at start (spy).
- [ ] Re-importing a DIFFERENT raw file (distinct sha256 — e.g. same data plus one extra
      ignored file) that parses to the same 12k events yields `eventsImported = 0`,
      `duplicates = 12k` (hash dedupe), and both imports listed. (Identical sha256 is
      rejected earlier by `findDuplicate` — separate test.)
- [ ] Day-precision duplicate counting: equal, larger, and smaller `play_count` re-imports
      each increment `duplicates` by 1; only the larger one changes the stored row.
- [ ] Duplicate upload detection: same sha256+source → `findDuplicate` returns prior import.
- [ ] Day-precision upsert: re-import with larger `play_count` updates the row (MAX policy);
      smaller does not.
- [ ] Fake parser throwing mid-file → import `failed`, its events fully removed, raw file kept,
      error_json code preserved.
- [ ] Zero-match zip → `ERR_NO_RECOGNIZED_FILES` with entry names + didYouMean when the fake
      "other" parser matches.
- [ ] Deletion: events gone (cascade), `identity.gc()` called, raw dir removed; delete during
      `parsing` rejected.
- [ ] Warning details capped at 1,000; counters still exact at 5,000 warnings.

## Validation

`npm test` (suite `src/importers/framework.test.ts` with temp DB + fake parsers), lint,
typecheck. Include one test using a real small zip through issue 08's iterator.

## Dependencies

03-config-and-paths, 04-sqlite-and-migrations, 05-core-normalization (types/hash),
08-zip-ingestion (implementation dependency — the zip path calls `iterateZipEntries`).
IdentityResolver is consumed via the interface above with a test fake (real one lands in 13).

## Non-goals

HTTP endpoints/multipart (14), real parsers (10-12), agg rebuild implementation (15), identity
GC implementation (13).

## Design References

DESIGN.md §6.1, §6.2, §6.5, §6.6, §6.7, §6.8, §14 (disk full, interrupted rows); ADR-002.
