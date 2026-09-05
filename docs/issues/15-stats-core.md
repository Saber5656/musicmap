# Title

Stats core: timezone bucketing, aggregate cache, overview and timeline queries

## Summary

Implement the metrics foundation per DESIGN.md §8: Intl-based month/year bucketing in the
configured display timezone, the `agg_monthly_artist` rebuild machinery with `agg_meta`
staleness detection, and the typed query functions `getOverview` and `getTimeline` with
golden-number tests against a hand-computed fixture library.

## Context

Every visualization reads through this layer. Correctness is defined by the mini-library
golden fixture: ~2k events across all 3 sources with known expected numbers (this issue
creates that fixture and its expectation tables; issues 16/28 extend them).

## Scope

- `src/core/bucketing.ts` (pure), `src/db/queries/stats.ts` (agg rebuild + overview +
  timeline), `fixtures/mini-library/` (events JSON + loader + expectations).
- Wire `onCompleted`/deletion/timezone-change rebuild hooks (replacing 14's no-op).

## Detailed Requirements

1. `src/core/bucketing.ts`:
   ```ts
   monthBucket(utcIso: string, timeZone: string): string   // 'YYYY-MM'
   yearBucket(utcIso: string, timeZone: string): string    // 'YYYY'
   hourOfDay(utcIso: string, timeZone: string): number     // 0-23
   ```
   Implemented with a cached `Intl.DateTimeFormat(..., { timeZone })` `formatToParts`
   (en-CA), no date libs. Invalid tz → throw `AppError('ERR_VALIDATION')` (from
   `src/core/errors.ts`, `details: { field: 'timeZone' }`). Pure + golden tests incl. DST
   boundaries (America/New_York 2021-03-14T06:30Z → '2021-03' hour 1; Asia/Tokyo fixed
   offset; UTC).
2. Agg rebuild (`rebuildAggregates(db, timeZone)`) — DESIGN §8.4's wording means SQL sums
   over TS-computed bucket keys (never SQLite `strftime`, which has no tz database):
   - One transaction: `DELETE FROM agg_monthly_artist`; scan `play_events` join `tracks`
     in 50k-row pages (`WHERE pe.id > :cursor ORDER BY pe.id LIMIT 50000`), computing
     `monthBucket` app-side; accumulate `(bucket, artist_id) → {plays += play_count,
     ms += coalesce(ms_played,0)}` in a Map, flushing INSERTs per page
     (`ON CONFLICT DO UPDATE SET plays = plays + excluded.plays, ms_played = ms_played +
     excluded.ms_played`); write `agg_meta` = `{ timezone, built_at, events_max_id }`.
   - `ensureAggFresh(db, timeZone)`: rebuild when `agg_meta.timezone !== timeZone` OR
     `events_max_id != MAX(play_events.id)` OR agg empty while events exist. Called at server
     boot and after import complete/delete (framework hooks) and by settings PATCH (issue 24
     calls it on tz change).
3. `getOverview(db, tz)` → DESIGN §9 `/api/overview` payload:
   ```ts
   { totals: { events, plays, minutes, durationCoveragePct, artists, tracks,
               earliestPlayedAt, latestPlayedAt },
     sources: [ { source, plays, minutes, earliest, latest,
                  monthlyCounts: [ { bucket, plays } ] } ],
     enrichment: { enabled, matched, pending, unmatched, failed } }
   ```
   Contracts: `monthlyCounts` includes EVERY month from that source's earliest to latest
   inclusive, with `plays: 0` rows for gap months (the UI never synthesizes months);
   `enrichment.enabled` resolves through the settings-default mechanism (missing row =
   `false`); counts read from `artist_enrichment`, zeros when empty.
   Also implement here (single owner, reused by every stats route):
   `getDisplayTimezone(db): string` in `src/db/queries/settings.ts` — reads
   `display.timezone` with system-tz fallback per §17.2 (issue 24 later builds the full
   settings registry around it).
4. `getTimeline(db, tz, { from?, to?, topN = 3, metric = 'plays' })` → §9 timeline payload:
   - `metric: 'plays' | 'minutes'` (invalid → `AppError('ERR_VALIDATION')` with the field in
     `details`); `topN` 1-10; `from`/`to` format `YYYY-MM` (validated); when omitted they
     default to the library's earliest/latest month in `tz`; empty library → `{ buckets: []
     }`.
   - Buckets: every month between max(from, earliest) and min(to, latest) INCLUSIVE, zero
     months included with `plays: 0` (gap honesty).
   - Per bucket: `plays`, `minutes` (from agg sums), `durationCoverage` defined EXACTLY as
     `SUM(CASE WHEN ms_played IS NOT NULL THEN play_count ELSE 0 END) / SUM(play_count)`
     over the bucket's events (single GROUP BY query over the whole range, not per bucket),
     `topArtists` (topN by chosen metric from agg, ties by name ASC), and `topTracks`
     (topN per bucket by metric from events JOIN tracks — single GROUP BY query over the
     range; shape `{ trackId, title, artistName, plays, minutes }`, per DESIGN §9).
   - Deterministic ordering everywhere (ties: name ASC, then id ASC).
5. Fixture `fixtures/mini-library/` (concrete contracts):
   ```ts
   // events.json: CanonicalPlayEvent[] (the issue-05 type, serialized as-is)
   // load.ts:
   export function loadMiniLibrary(db: Database): void;   // imports via the real framework
   // expected.ts:
   export const EXPECTED: {
     [tz in 'Asia/Tokyo' | 'UTC']: {
       overview: { events: number; plays: number; minutes: number;
                   durationCoveragePct: number; artists: number; tracks: number };
       timelineBuckets: Record<string /* 'YYYY-MM' */, { plays: number; minutes: number;
                   durationCoverage: number }>;           // ≥ 6 chosen buckets
       topArtists: Record<string /* bucket */, Array<{ name: string; plays: number }>>;
     }
   };
   ```
   - `events.json`: ~2,000 canonical events, 3 sources, spanning 2018-01..2021-12 with
     deliberate: an empty month, a YTM-only month (durationCoverage 0), a day-precision
     Apple batch, same-artist-cross-source rows, tz-sensitive events (23:30 JST month
     boundary). Every expectation includes the arithmetic in comments.
6. All SQL prepared; no string interpolation (repo rule).

## Acceptance Criteria

- [ ] Bucketing golden tests pass incl. DST + month-boundary JST cases.
- [ ] Rebuild is idempotent; incremental correctness: import → rebuild equals
      recompute-from-scratch (property test comparing two paths on the fixture).
- [ ] `ensureAggFresh` triggers on tz change, new import, deletion; no-ops otherwise
      (spy on rebuild).
- [ ] Overview + timeline golden numbers match `expected.ts` exactly for both timezones
      (incl. `topTracks` arrays and the day-precision durationCoverage weighting by
      `play_count`).
- [ ] Zero-month inclusion and durationCoverage per bucket verified; empty library returns
      empty buckets without error.
      (The 1M-event rebuild budget is measured in issue 32, not here.)

## Validation

`npm test` (suites `src/core/bucketing.test.ts`, `src/db/stats.test.ts`), lint, typecheck.

## Dependencies

04, 09 (framework for fixture load + hooks), 13 (identity), 14 (replaces its no-op
`onCompleted` wiring).

## Non-goals

Trends/wrapped/clock/discovery (16), HTTP routes (19/20 wire these functions), genre
aggregates (28 composes at query time).

## Design References

DESIGN.md §8.1-8.4 (normative), §9 (payload shapes), §15 (rebuild budget); ADR-006.
