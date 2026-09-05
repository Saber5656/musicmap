# Title

Artwork pipeline: release-group resolution, CAA fetch/cache, wrapped integration

## Summary

Implement bounded album-artwork enrichment per DESIGN.md §12.5: derive (artist, album)
candidates from wrapped top artists, resolve release-group MBIDs via MusicBrainz, fetch
front-500 covers from Cover Art Archive to the artwork disk cache, serve them via
`GET /api/artwork/:id`, and surface `artworkId` in wrapped payloads.

## Context

Artwork is polish with strict bounds (≤ 200 rows, top-5 artists per wrapped year only).
It reuses issue 25's clients and rides issue 26's runner cadence — a separate low-priority
queue phase that runs only after artist enrichment completes.

## Scope

- `src/enrichment/artwork.ts` (candidate builder + fetch phase), extend
  `src/db/queries/enrichment.ts` (artwork rows), `src/server/routes/artwork.ts`,
  wrapped payload wiring (issue 16's LEFT JOIN becomes live).

## Detailed Requirements

1. Candidate derivation `buildArtworkCandidates(db)` (per DESIGN §12.1/§12.5 — one album
   per artist-year, the one wrapped renders):
   - For each year in `getWrappedYears`, take that year's top-5 artists (metric `plays`);
     for each, its most-played non-null album string that year (skip artists with no album
     data — YTM/Apple-only artists usually).
   - Upsert into `artwork (artist_norm, album_norm, status='pending')` respecting the UNIQUE
     key; global cap: stop inserting when total artwork rows would exceed 200 (deterministic
     order: year DESC, rank ASC — recent years win).
2. Fetch phase — extends issue 26's runner behind its shipped status contract (§12.4):
   when the artist queue drains, the runner sets `phase: 'artwork'`, runs
   `buildArtworkCandidates`, processes pending artwork rows, and only then transitions
   `running → completed`; `artworkCounts { pending, fetched, not_found, failed }` appears in
   `/api/enrichment/status`; the same FSM/pause/resume/disable rules apply mid-phase. MB
   search counts against the 1,100 ms limiter, CAA against its own:
   - Per pending row: `searchReleaseGroup(artistDisplayName, albumDisplayName)` (denormalize
     names from any track carrying that album) → accept top hit score ≥ 90 → store
     `release_group_mbid`; else status `not_found`.
   - `fetchFrontCover(mbid)` → `fetched`: write bytes to `<dataDir>/artwork/<rowId>.<ext>`
     (extension from the validated content-type; the stored relative `file_path` is
     authoritative), set fetched_at. CAA 404 → status `not_found`; too-large/wrong
     content-type/transport failure → status `failed` (the §5 enum has no reason column —
     assert the reason via the client result in tests, not the schema).
   - Disk write via temp file + rename (no partial files); fs errors → `failed`.
3. Route `GET /api/artwork/:id`: bearer-protected (per issue 23 rule); looks up row status
   `fetched` → stream file with correct content-type + `Cache-Control: private, max-age=
   31536000, immutable`; the global security headers (issue 06 onSend hook, incl.
   `X-Content-Type-Options: nosniff`) apply — assert them on this route; any other
   status/unknown id → 404 envelope. Path resolution ONLY from
   `paths(dataDir).artworkDir + basename(file_path)` — never from client input beyond the
   integer id (zod int coercion).
4. Wrapped wiring: `getWrapped` `topArtists[].artworkId` resolves via
   `artwork WHERE artist_norm = ? AND album_norm = ? AND status='fetched'` for the artist's
   top album that year (matches candidate derivation logic — extract the shared
   "top album for artist-year" SQL into one query function used by both).
5. Deletion: issue 24's delete-data already truncates artwork rows + files (verify it covers
   files written here — add test).

## Acceptance Criteria

- [ ] Candidate builder on an extended mini fixture: exact expected (artist, album, year)
      set; 200-row cap enforced with recent-year priority (synthetic overflow test).
- [ ] Fetch phase (MockAgent): pending → fetched with file on disk (bytes match mock);
      not_found (CAA 404); too_large → failed; MB no-match → not_found without CAA call
      (call-count assert).
- [ ] Runner integration: artwork phase starts only after artist phase completed; pause
      applies mid-phase; restart resumes pending artwork rows.
- [ ] Route: fetched → 200 with content-type + cache + security headers;
      pending/failed/unknown → 404; 401 without token; non-integer id (`/api/artwork/abc`)
      → 400; encoded traversal (`/api/artwork/%2e%2e%2f1`) never touches disk and returns
      400 or 404 (fs spy).
- [ ] Wrapped payload now carries artworkId for the fixture's fetched row; null elsewhere
      (golden update).
- [ ] delete-data removes files created here (fs assert).

## Validation

`npm test`, lint, typecheck. Manual (optional, respectful): one real CAA fetch locally,
screenshot of wrapped card with artwork in PR.

## Dependencies

23-wrapped-view (render slot + blob loader), 25-musicbrainz-client, 26-enrichment-runner,
24-settings-and-consent (delete-data flow covers artwork files — verified here).

## Non-goals

Artwork on timeline/trends (v2), track-level art, retina 1200px variants, public (tokenless)
image serving.

## Design References

DESIGN.md §12.5 (normative), §5 (artwork DDL), §9 (route), §11.3 (top-artist thumbs);
ADR-003 (bounds, allowlist).
