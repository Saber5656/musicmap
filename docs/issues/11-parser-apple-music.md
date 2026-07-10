# Title

Apple Music parser: Play Activity CSV + Play History Daily Tracks fallback

## Summary

Implement the `apple_music` ImportSourceParser handling both `Apple Music Play Activity.csv`
(event-level, primary) and `Apple Music - Play History Daily Tracks.csv` (day-level fallback),
with header-alias tolerance, playback-row filtering, duration clamping, timestamp
reconstruction, and PII dropping — per `docs/research/apple-music-export.md` and DESIGN.md
§6.4.

## Context

Apple exports have the messiest schema drift (column renames across vintages) and the only
in-export genre labels (kept as rawMeta for possible v2 use; NOT used for v1 genre features,
which are MusicBrainz-only for consistency — ADR-003).

## Scope

- `src/importers/apple.ts` + fixtures `fixtures/exports/apple/` + tests.
- Add dependency `csv-parse`.

## Detailed Requirements

1. `matches(entryName, peek)`:
   - Play Activity: basename contains `Play Activity` (case-insensitive) and `.csv`, OR any
     `.csv` whose peek header row contains (after BOM strip) BOTH a title alias and
     `Play Duration Milliseconds`.
   - Daily Tracks: basename contains `Play History Daily Tracks`, OR header contains
     `Track Description` AND `Date Played`.
   - Must reject Spotify/YTM peeks.
2. CSV handling (`csv-parse` streaming): `{ bom: true, columns: true, relax_column_count:
   true }`; per-record size guard 10 MiB (T6) via `max_record_size` — a csv-parse
   max-record error is FATAL: throw `AppError('ERR_SCHEMA_MISMATCH', { reason:
   'csv_record_too_large' })`; a single cell over 64 KiB → `skip:invalid` for that row only
   (not fatal). Parsing stays streaming with bounded memory.
3. **Play Activity mapping** (header aliases in a table `HEADER_ALIASES`):
   | Canonical | Column (aliases) | Rule |
   |---|---|---|
   | trackTitle | `Content Name` \| `Song Name` \| `Original Title` (first present, in this order) | trim; empty → `skip:invalid` |
   | artistName | `Artist Name` | trim; empty → `skip:invalid` |
   | playedAt | `Event Start Timestamp`, else `Event End Timestamp` minus duration, else `Event Received Timestamp` | parse ISO; all absent/invalid → `skip:invalid`; normalize to UTC `...Z`; precision `'second'` |
   | msPlayed | `Play Duration Milliseconds` | parse int; negative → 0 + `W_NEGATIVE_MS`; if `Media Duration In Milliseconds` present and > 0, clamp to it; absent → omit |
   | playCount | — | 1 |
   | rawMeta.event_type | `Event Type` | ≤ 64 chars |
   | rawMeta.utc_offset_seconds | `UTC Offset In Seconds` | int; stored, unused in v1 (§8.2) |
   | rawMeta.apple_genre | `Genre` | ≤ 64 chars |
   - Row filter BEFORE mapping — deterministic algorithm, applied in this order:
     1. `Media Type` present and ≠ `AUDIO` (case-insensitive) → `skip:non_audio`.
     2. `Event Type` in the known non-playback set `{LYRIC_DISPLAY}` → `skip:non_audio`.
     3. File-level mode decision (no second pass; single-buffer heuristic): buffer the
        first 1,000 records before emitting anything — if any of them has
        `Event Type = PLAY_END`, mode = `play_end_only`, else mode = `all_playback`; the
        mode is then fixed for the remainder of the file. In `play_end_only` mode, keep
        only `PLAY_END` rows (others → `skip:non_audio`); in `all_playback` mode, keep rows
        (incl. `PLAY_START`/unknown types) with `Play Duration Milliseconds > 0`, else
        `skip:invalid`.
   - **PII denylist** (never copied): `Apple Id Number`, `Client IP Address`,
     `Device Identifier`, `Metrics Client Id`, `Metrics Bucket Id`, `Build Version`.
4. **Daily Tracks mapping**:
   | Canonical | Column | Rule |
   |---|---|---|
   | artistName + trackTitle | `Track Description` | split on FIRST `" - "`; no separator → whole string is title, artist `Unknown Artist` + `warning W_NO_ARTIST_SPLIT` |
   | playedAt | `Date Played` (`YYYYMMDD`) | → `YYYY-MM-DDT00:00:00Z` (DESIGN §5 overrides the research doc's earlier local-noon idea — no local-time reconstruction); invalid → `skip:invalid` |
   | precision | — | `'day'` |
   | playCount | `Play Count` | int ≥ 1; absent/invalid → 1 |
   | msPlayed | `Play Duration Milliseconds` | total for the day; negative → 0 |
5. Missing required headers (no alias matched) → `AppError('ERR_SCHEMA_MISMATCH')` with
   `details: { expected: [...aliases], found: [...actual headers] }` (fatal).
6. File-selection rule (normative): implement the optional parser method
   `selectFiles(matchedNames)` (issue 05 contract; called by the issue 09 framework) — when
   BOTH Play Activity and Daily Tracks files matched, return only the Play Activity file(s)
   in `use` plus one warning `{ code: 'W_DAILY_TRACKS_SKIPPED', detail: <skipped names> }`
   (cross-file dedupe is impossible: different precisions produce different hashes, so
   importing both would double-count). Daily Tracks parses only when no Play Activity file
   is present.
7. Fixtures: play-activity modern vintage (Song Name), older vintage (Content Name), rows
   covering LYRIC_DISPLAY skip, negative duration, clamping, missing start timestamp
   (derived from end − duration), video row (`Media Type: VIDEO`), ja titles; daily-tracks
   file with split/no-split descriptions; oversized-cell and oversized-record fixtures
   (T6); a zip containing the Apple folder structure + both CSVs.

## Acceptance Criteria

- [ ] Both vintages map to exact expected events (literal expectations).
- [ ] Filter matrix tested: AUDIO/VIDEO × PLAY_END/LYRIC_DISPLAY/unknown × duration
      present/absent behaves per rules above, in BOTH file-level modes (`play_end_only`
      and `all_playback`, incl. the 1,000-record mode-decision buffer boundary).
- [ ] T6: a >64 KiB cell yields exactly one `skip:invalid` and parsing continues; a >10 MiB
      record aborts with `ERR_SCHEMA_MISMATCH` (`csv_record_too_large`); RSS stays bounded
      on a generated large file (same method as issue 10).
- [ ] Timestamp reconstruction (end − duration) verified to the second.
- [ ] Clamp: `Play Duration` 10× media duration → clamped; `W_NEGATIVE_MS` counted.
- [ ] Daily Tracks: split, no-split, playCount, day precision (hash stability via issue 05
      day rule) verified.
- [ ] Archive with both files: `selectFiles` returns only Play Activity +
      `W_DAILY_TRACKS_SKIPPED` warning (framework-level test through issue 09).
- [ ] PII columns never appear in any yielded event.
- [ ] Missing-headers file → `ERR_SCHEMA_MISMATCH` with expected/found lists.

## Validation

`npm test` (suite `src/importers/apple.test.ts`), lint, typecheck. Note real-export
verification as a checklist item in the PR (K1 in ISSUE_PLAN §8).

## Dependencies

05-core-normalization, 08-zip-ingestion (test-level), 09-import-framework (test-level, for
the `selectFiles` framework test).

## Non-goals

Library/Recently Played files, Apple genre → genre features (MusicBrainz only in v1),
persistence.

## Design References

DESIGN.md §6.4, §8.2 (offset unused), §13.2 T6/T10; docs/research/apple-music-export.md
(normative); ADR-002, ADR-003.
