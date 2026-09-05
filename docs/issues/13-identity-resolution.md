# Title

Artist/track identity resolution: upserts, native-id capture, orphan GC

## Summary

Implement the IdentityResolver used by the import framework: normalized-key artist/track
upserts with in-memory caching, native-id/album backfill, and the orphan garbage collector run
after import deletion — per DESIGN.md §7.

## Context

Identity is what merges "the same song" across Spotify/Apple/YTM into one track row, powering
all cross-source stats. Rules are deliberately conservative (exact normalized match only);
fancier merging is a documented v1 non-goal.

## Scope

- `src/db/queries/identity.ts` + tests.

## Detailed Requirements

1. Interface (declared in issue 09's framework types — import it from there, do not
   redeclare):
   ```ts
   interface IdentityResolver {
     resolve(e: CanonicalPlayEvent): { artistId: number; trackId: number;
                                       artistCreated: boolean; trackCreated: boolean };
     gc(): { tracksDeleted: number; artistsDeleted: number };
     resetCache(): void;  // called by framework at import start
   }
   createIdentityResolver(db: Database): IdentityResolver;
   ```
   The `artistCreated`/`trackCreated` flags are true only when this call performed the
   INSERT — the framework sums them into `ImportReport.artistsCreated`/`tracksCreated`
   (§6.6).
2. `resolve` semantics (must run inside the framework's flush transaction). Display-string
   policy (uniform): stored display `name`/`title` are the parser-provided raw strings with
   only leading/trailing whitespace trimmed; Unicode content otherwise round-trips exactly.
   - `key = norm(e.artistName)`; lookup cache → `SELECT id FROM artists WHERE norm_name = ?`
     → `INSERT INTO artists(name, norm_name) VALUES (display artistName, key)`.
   - `tkey = normTitle(e.trackTitle)`; same pattern against
     `tracks(artist_id, norm_title)`; insert stores display title, `album`, and native ids.
   - Backfill on existing track (single UPDATE, only when the column IS NULL):
     `album`, `spotify_track_uri`, `youtube_video_id` from the event when present.
     Display `name`/`title` are NEVER updated (first-seen wins — deterministic given import
     order; documented).
   - Caches: `Map<norm_name, id>` and `Map<artistId + ' ' + norm_title, id>`; must be
     correct across flush transactions within one import (ids are stable once inserted).
3. `gc()` (runs inside the deletion transaction, DESIGN §7.3): delete tracks with no events,
   then artists with no tracks; `artist_enrichment`/`artist_genres` rows disappear via
   `ON DELETE CASCADE`; returns counts. Also delete `agg_monthly_artist` rows for deleted
   artists (cascade covers it — verify).
4. Edge rules (tests are normative):
   - Same artist different casing/width (`YOASOBI` vs `ＹＯＡＳＯＢＩ`) → one artist row,
     display name = first seen.
   - Same title with/without `(feat. …)` → one track (normTitle strip).
   - Same normalized title under different artists → distinct tracks.
   - Empty-after-norm artist/title never reaches resolve (parsers guarantee; resolver throws
     `ERR_INTERNAL` if it happens — defensive assertion).
   - Unicode ja titles round-trip exactly in display fields.

## Acceptance Criteria

- [ ] Cross-source merge test: Spotify + YTM + Apple events for the same song (varying case,
      fullwidth, feat-suffix) produce 1 artist, 1 track, 3 events; track has BOTH
      spotify_track_uri and youtube_video_id after backfill.
- [ ] Backfill never overwrites non-NULL values (test with conflicting albums).
- [ ] GC removes exactly the orphans and their enrichment cascades after deleting one of two
      imports sharing an artist (shared artist survives).
- [ ] Created-flag counting: a batch with 3 new artists, 5 new tracks, and repeats reports
      exactly those totals via the flags.
- [ ] 50k-event resolve benchmark inside one tx stays under 2 s locally (loose smoke bound,
      logged not asserted in CI).
- [ ] All variable values bind via prepared-statement parameters; no interpolated or
      concatenated SQL — lint/review.

## Validation

`npm test` (suite `src/db/identity.test.ts` with temp DB + real migrations), lint, typecheck.

## Dependencies

04-sqlite-and-migrations, 05-core-normalization, 09-import-framework (declares the
`IdentityResolver` interface this issue implements).

## Non-goals

Multi-artist splitting, MBID-based merging (enrichment does not merge identities in v1),
fuzzy matching, renames UI.

## Design References

DESIGN.md §7 (normative), §5 (tables), §6.8 (tx context); ADR-002.
