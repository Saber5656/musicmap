# Title

Stats extended: trends series, wrapped aggregates, discovery, listening clock

## Summary

Implement the remaining query functions per DESIGN.md §8/§11: `getTrends` (artist and genre
series with `__other__`/`__unknown__` buckets), `getWrappedYears`/`getWrapped` (all §11.3
cards), first-play discovery, and the listening clock — extending the mini-library golden
expectations.

## Context

These feed the trends view (22) — including its genre-mode gating — and the wrapped view
(23); the genre map (28) must stay numerically consistent with them. Genre data comes from
`artist_genres` (enrichment); functions must behave correctly when that table is empty.

## Scope

- Extend `src/db/queries/stats.ts`; extend `fixtures/mini-library/expected.ts` (add a small
  seeded `artist_genres` set to the fixture loader for genre-mode tests).

## Detailed Requirements

1. `getTrends(db, tz, { mode: 'artist'|'genre', from?, to?, topN = 10, metric = 'plays' })`
   → §9 shape (`metric: 'plays'|'minutes'`; minutes ranks/sums use `ms_played` sums —
   duration-null events contribute 0):
   - Buckets: all months in range (zero-filled); `from`/`to` omitted → earliest/latest play
     month in `tz`; empty library → `{ buckets: [], series: [] }`.
   - `mode 'artist'`: top-N artists by total metric over the whole range (from agg); series
     values per bucket from agg; all other artists summed into `{ id: -1, name: '__other__' }`
     (always last in `series`).
   - `mode 'genre'`: artist weights fan out to genres via `artist_genres` **fractionally**
     (an artist with k genres contributes weight/k to each — keeps bucket sums equal to
     artist-mode totals; normative). Artists with zero genres contribute to
     `{ id: -2, name: '__unknown__' }`. Top-N genres + `__other__` + `__unknown__` (order:
     series desc by total, then `__other__`, then `__unknown__`).
   - Empty `artist_genres` + mode genre → return `{ series: [__unknown__ only] }` (route 22
     decides UI gating; no error here).
2. `getWrappedYears(db, tz)` → `{ years: number[] }` ascending, only years with ≥ 1 play.
3. `getWrapped(db, tz, year)` → exact payload type (normative — §11.3 cards):
   ```ts
   interface WrappedPayload {
     year: number;
     totals: { plays: number; minutes: number; durationCoveragePct: number;
               distinctArtists: number; distinctTracks: number };
     topArtists: Array<{ artistId: number; name: string; plays: number; minutes: number;
                         artworkId: number | null }>;       // 5; artwork LEFT JOIN via the
                                                            // artist's most-played album
                                                            // this year (rows exist only
                                                            // after issue 27; else null)
     topTracks: Array<{ trackId: number; title: string; artistName: string;
                        plays: number }>;                   // 5
     topGenres: Array<{ genreId: number; name: string; weight: number }>;  // ≤3; REAL
                        // genres only — never __unknown__/__other__; [] when contributing
                        // artists have no artist_genres rows
     clock: number[];                                       // 24 ints; second-precision only
     firsts: { firstPlay: PlayRef | null; lastPlay: PlayRef | null;
               busiestDay: { date: string; plays: number } | null;
               maxSameTrackDay: { date: string; trackTitle: string; artistName: string;
                                  plays: number } | null }; // null only possible when the
                                                            // year has exclusively
                                                            // day-precision events (clock
                                                            // empty) — dates are local tz
     discoveries: { count: number;
                    artists: Array<{ artistId: number; name: string; plays: number;
                                     firstPlayedAt: string }> };  // top 3 by in-year metric
     sources: Array<{ source: Source; plays: number; sharePct: number }>;
   }
   interface PlayRef { playedAt: string; trackTitle: string; artistName: string }
   ```
   - ANY year with zero plays — outside the data range OR a gap year between populated
     years — throws `ERR_NOT_FOUND` (route maps to 404).
4. Determinism: all ties broken by name ASC then id ASC (same rule as 15).
5. Golden expectations: extend fixture with a seeded genre set (e.g. artist A: [rock],
   artist B: [rock, shoegaze] → fractional check 0.5/0.5) and hand-computed: one trends
   payload (artist mode, 2019 full year, topN=2), one genre-mode payload, wrapped 2019
   (every card field), clock array, discoveries.

## Acceptance Criteria

- [ ] Trends artist mode: bucket sums equal overview plays for the same range (invariant
      test on the whole fixture).
- [ ] Genre fractional fan-out: per-bucket sum(series incl. __unknown__/__other__) equals
      artist-mode sum (invariant + golden).
- [ ] Wrapped 2019 golden matches exactly, including clock excluding day-precision rows,
      discoveries using global first-play, and `topGenres` excluding unknown weight (golden
      case mixes genre-tagged and untagged artists).
- [ ] `metric = 'minutes'` goldens for trends and wrapped top lists (duration-null events
      contribute 0 and ranking differs from plays in the fixture).
- [ ] `getWrapped(1999)` → ERR_NOT_FOUND; a zero-play gap year INSIDE the data range also →
      ERR_NOT_FOUND; `getWrappedYears` matches fixture years (gap year absent).
- [ ] Genre mode with empty enrichment returns only `__unknown__` (no crash); empty library
      returns empty buckets/series.
- [ ] All new SQL prepared statements; deterministic ordering tests (run twice, deep-equal).

## Validation

`npm test` (extended `src/db/stats.test.ts` suites), lint, typecheck.

## Dependencies

15-stats-core.

## Non-goals

HTTP routes (22/23), genre-map node/edge computation (28), artwork fetching (27 — only the
LEFT JOIN contract is honored here).

## Design References

DESIGN.md §8.1 (metrics), §11.2 (series semantics), §11.3 (card inventory), §9 (payloads),
§14 (zero-genre artists → `__unknown__` row).
