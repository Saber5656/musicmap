# Title

Synthetic export fixture generator (all 3 sources, up to 1M events)

## Summary

Implement `scripts/generate-fixtures.ts`: a seeded, deterministic generator producing
realistic Spotify/Apple/YouTube-Music export files (and ZIP variants) at configurable scales,
used by E2E (30) and performance validation (32).

## Context

Real exports cannot be committed (privacy). The committed hand-written parser fixtures are
tiny; E2E needs a mid-size realistic library and perf needs 1M events — both must be
reproducible byte-for-byte from a seed so failures are debuggable.

## Scope

- `scripts/generate-fixtures.ts` + `src/testing/synth.ts` (shared generation lib, importable
  by tests); output to `fixtures/generated/` (gitignored, per issue 01).

## Detailed Requirements

1. CLI: `npx tsx scripts/generate-fixtures.ts --scale small|medium|perf --seed 42
   --out fixtures/generated`. "Events" below means EXPECTED CANONICAL IMPORTED events
   (post-skip, post-dedupe) — generated raw rows are higher (skipped rows, YTM non-music
   noise at 3× the YTM music count, planted duplicates):
   | scale | canonical events total | per-source split (spotify/apple/ytm) | span |
   |---|---|---|---|
   | small | ~5,000 | 60% / 25% / 15% | 3 years |
   | medium | ~120,000 | 60% / 25% / 15% | 8 years |
   | perf | 1,000,000 | 60% / 25% / 15% | 12 years |
   The manifest (below) records the exact per-source expectations computed during
   generation, so tests assert against the manifest, never against these approximations.
2. Model (seeded mulberry32; NO `Math.random`): a synthetic "life" — 40-400 artists with
   popularity curves rising/falling over eras (so timeline/trends/map look real), 5-30
   tracks per artist (weighted), daily listening sessions (evening-skewed hour distribution
   for clock realism), per-source coverage windows (spotify oldest, apple middle, ytm
   recent, with overlaps for dedupe/merge realism) and one deliberate 2-month global gap.
   ~2% of events reference the same (artist, track) across sources with normalization
   variants (case/fullwidth/feat suffixes) to exercise identity merging.
3. Outputs per source, matching the REAL formats exactly as specified in the "Archive
   layout" / file-location sections of `docs/research/*.md` (those headings are normative
   for folder names per source/locale) and consumed by issues 10-12's `matches`/`parse`
   without special cases. ZIP variants are written with `yazl` (devDep, allowlisted in
   DESIGN §4 for fixture writing only):
   - Spotify: `Streaming_History_Audio_<years>_<n>.json` files (~10k events each) + a few
     podcast rows + planted PII values (see manifest below) exercising the drop rules; ZIP
     variant using the research-doc folder name `Spotify Extended Streaming History/`.
   - Apple: `Apple Music Play Activity.csv` (modern headers) incl. LYRIC_DISPLAY rows,
     negative durations, missing-start rows, AND a small
     `Apple Music - Play History Daily Tracks.csv` in the SAME ZIP — the manifest expects
     Play Activity events only plus a `W_DAILY_TRACKS_SKIPPED` warning (issue 11 selectFiles
     rule); the loose-file variants are emitted separately for single-file tests.
   - YTM: `watch-history.json` (en) and a ja variant (ja affixes + localized zip folder
     names per the research doc), ad entries, deleted entries, non-music YouTube rows
     (~3× music count).
4. Manifest schema (exact; computed DURING generation — ground truth for E2E/perf
   assertions without re-parsing):
   ```ts
   interface FixtureManifest {
     seed: number; scale: 'small'|'medium'|'perf'; generatedWith: string; // package version
     perSource: Record<Source, { files: string[]; rowsTotal: number;
       expectedEvents: number; expectedDuplicates: number;
       expectedSkips: Record<string, number>;    // reason → count
       expectedWarnings: string[] }>;            // e.g. ['W_DAILY_TRACKS_SKIPPED']
     expectedDistinctArtists: { min: number; max: number };
     plantedPiiMarkers: { ip: string; email: string; deviceId: string; username: string };
       // stable literals placed ONLY inside raw export files; issue 31 greps the DB/logs
       // for exactly these values
   }
   ```
5. Determinism: same seed+scale → byte-identical outputs (stable ordering, fixed timestamp
   formatting); generation time for `perf` ≤ 60 s.
6. `src/testing/synth.ts` exports the event-stream generator for direct in-test use
   (issue 15's fixture loader may migrate to it later — out of scope here).

## Acceptance Criteria

- [ ] `small` outputs parse through the real registry parsers with zero `invalid` skips
      beyond the deliberately planted ones; counts match `manifest.json` exactly
      (integration test in this issue).
- [ ] Determinism test: two runs (seed 42) → identical sha256 per file.
- [ ] ja YTM variant round-trips through issue 12's parser (affix stripping verified against
      manifest counts).
- [ ] Cross-source merge realism: importing all three `small` sources yields distinct
      artists within manifest min/max (identity merging worked).
- [ ] `perf` scale generates 1M events ≤ 60 s locally (timed, reported in PR).
- [ ] No `Math.random`/`Date.now` in generation paths (grep + lint).

## Validation

`npm test` (suite `scripts/generate-fixtures.test.ts` runs `small` end-to-end through the
import framework), lint, typecheck.

## Dependencies

01-project-scaffold, 05-core-normalization (generation lib), and — for the integration test
that parses `small` through the real pipeline — 09, 10, 11, 12, 13 (wave order guarantees
availability).

## Non-goals

Committing generated outputs (gitignored), demo-mode UI, replacing the hand-written golden
mini-library (it stays authoritative for stats numbers).

## Design References

DESIGN.md §16 (fixture strategy), §15 (perf inputs); docs/research/*.md (format fidelity).
