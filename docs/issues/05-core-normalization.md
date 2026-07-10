# Title

Core domain types, name normalization, and event hash

## Summary

Implement the pure core layer: canonical types (`CanonicalPlayEvent`, `ParsedRow`,
`ImportSourceParser`, `ImportReport`, error codes), the `norm`/`normTitle` normalization
pipeline, and the idempotency `eventHash` function — exactly per DESIGN.md §6.4, §6.5, §7.1.

## Context

Every parser, the import framework, identity resolution, and dedupe correctness hang off these
functions. They are pure (no I/O) and must be exhaustively table-tested, including Japanese
text, because determinism here defines idempotent imports.

## Scope

- `src/core/types.ts`, `src/core/normalize.ts`, `src/core/hash.ts`, `src/core/errors.ts`.
- Golden-table unit tests.

## Detailed Requirements

1. `types.ts`: implement the §6.4 contract (`Source`, `CanonicalPlayEvent`, `ParsedRow`,
   `SkipReason`, `ImportSourceParser` incl. two optional methods:
   `hintFor(entryName: string): string | undefined` — issue 12 uses it for the HTML-format
   hint — and `selectFiles(matchedNames: string[]): { use: string[]; warnings: { code:
   string; detail: string }[] }` — issue 11 uses it to suppress redundant Apple files; the
   framework, issue 09, calls both) plus these project shared types:
   ```ts
   interface ParseContext {
     log: { debug(msg: string): void; warn(msg: string): void };  // minimal, no pino dep
     onBytes?: (deltaBytes: number) => void;                       // DELTA bytes consumed
   }
   type Metric = 'plays' | 'minutes';
   ```
   and `ImportReport` per §6.6. `rawMeta` values are primitives only
   (`string | number | boolean`) and parsers may emit ONLY the whitelisted keys their issue
   tables define — never raw source objects. No runtime deps.
2. `errors.ts`: `export const ErrorCodes = { ... } as const` containing every code in §6.7 and
   §9 (`ERR_UPLOAD_TOO_LARGE`, `ERR_ARCHIVE_ENCRYPTED`, `ERR_ARCHIVE_BOMB`,
   `ERR_ARCHIVE_CORRUPT`, `ERR_NO_RECOGNIZED_FILES`, `ERR_SCHEMA_MISMATCH`, `ERR_DISK_FULL`,
   `ERR_DB`, `ERR_DB_CORRUPT`, `ERR_MIGRATION_FAILED`, `ERR_INTERNAL`, `ERR_INTERRUPTED`,
   `ERR_DUPLICATE_UPLOAD`, `ERR_VALIDATION`, `ERR_UNAUTHORIZED`, `ERR_FORBIDDEN_HOST`,
   `ERR_NOT_FOUND`, `ERR_CONFLICT`, `ERR_ENRICHMENT_DISABLED`, `ERR_DATA_DIR_INVALID`) and an
   `AppError extends Error { code, details? }` class.
3. `normalize.ts`:
   - `norm(s: string): string` implementing §7.1 steps 1-6 exactly in DESIGN's order:
     (1) NFKC → (2) lowercase (locale-independent) → (3) NFKD + strip `\p{M}` →
     (4) strip zero-width chars (U+200B..U+200D, U+FEFF) → (5) map `’‘` → `'`, `“”` → `"`,
     `–—―` → `-` → (6) collapse `\s+` to single space + trim. The golden tests below are
     normative for edge cases.
   - `normTitle(s: string): string` = apply `norm(s)` first, then strip ONE trailing group
     matching `/\s*[(\[](feat\.|ft\.|with\s|prod\.)[^)\]]*[)\]]\s*$/i` from the normalized
     string, then trim. The golden tests below are normative for edge cases.
   - Both exported pure; no locale-dependent APIs (`toLowerCase()` without locale arg).
4. `hash.ts`: `eventHash(e: CanonicalPlayEvent): string` per §6.5:
   - `second`: `sha256(source + '|' + playedAt + '|s|' + norm(artistName) + '|' +
     normTitle(trackTitle) + '|' + (msPlayed ?? '-'))`
   - `day`: `sha256(source + '|' + playedAt + '|d|' + norm(artistName) + '|' +
     normTitle(trackTitle))`
   - hex lowercase; use `node:crypto`. (`src/core` may import node builtins; the "pure" rule
     bans project-layer imports, not stdlib.)
5. Golden test tables (minimum cases; add more freely):
   | input | norm |
   |---|---|
   | `"  Beyoncé "` | `beyonce` |
   | `"ＹＯＡＳＯＢＩ"` (fullwidth) | `yoasobi` |
   | `"米津玄師"` | `米津玄師` (CJK preserved) |
   | `"Ado　－　うっせぇわ"` (ideographic space, fullwidth dash) | `ado - うっせぇわ` |
   | `"A​B"` | `ab` |
   | `"“Quoted”"` | `"quoted"` |
   normTitle: `"Song (feat. X)"` → `song`; `"Song (Live)"` → `song (live)` (NOT stripped);
   `"Song [prod. Y]"` → `song`; `"(feat. X) Song"` → `(feat. x) song` (leading not stripped).
   eventHash: fixed vectors — assert exact sha256 hex for 2 events (compute once, freeze in
   test), plus properties: msPlayed change flips second-precision hash; playCount never
   affects hash; day-precision ignores msPlayed.

## Acceptance Criteria

- [ ] All golden tables pass; functions are deterministic across runs and platforms.
- [ ] `norm`/`normTitle`/`eventHash` have 100% line coverage (vitest coverage on these files).
- [ ] No imports from other `src/` layers (lint rule from issue 01 stays green).
- [ ] Error-code const covers every `ERR_*` string appearing anywhere in `docs/DESIGN.md`
      and `docs/issues/*.md` (verify with `rg -o 'ERR_[A-Z_]+' docs | sort -u` — paste the
      diff-check in the PR).
- [ ] `rawMeta` type rejects non-primitive values at compile time (type-level test with
      `@ts-expect-error`).

## Validation

`npm test`, `npm run lint`, `npm run typecheck`. Include the golden tables in the PR diff for
review.

## Dependencies

01-project-scaffold.

## Non-goals

Persistence (04/13), parser implementations (10-12), multi-artist splitting or romaji/kana
unification (documented v1 limitation, DESIGN §7.1).

## Design References

DESIGN.md §6.4, §6.5, §6.6, §6.7, §7.1; ADR-002.
