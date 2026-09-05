# Title

YouTube Music parser: Google Takeout watch-history.json

## Summary

Implement the `youtube_music` ImportSourceParser: stream-parse Takeout `watch-history.json`,
select music rows (`header == "YouTube Music"` or `music.youtube.com` titleUrl), strip
localized "Watched" affixes (en/ja), derive artists from `" - Topic"` channels, exclude ads
and deleted videos, and emit duration-less events — per
`docs/research/youtube-music-takeout.md` and DESIGN.md §6.4.

## Context

YTM is the lowest-fidelity source: timestamps only, no durations, localized strings. Honesty
rules apply: no fabricated durations; unparsable rows are counted, never guessed silently.

## Scope

- `src/importers/youtube.ts` + fixtures `fixtures/exports/youtube/` + tests.
- Uses `stream-json` (added in issue 10).

## Detailed Requirements

1. `matches(entryName, peek)`:
   - Name: basename is `watch-history.json` OR ja `再生履歴.json` OR any `.json` under a path
     segment `history`/`履歴` whose peek sniffs to a Takeout array. Content sniff: trimmed
     starts `[` and contains `"titleUrl"` or `"activityControls"` or `"header"`.
   - Must reject Spotify JSON (contains `"ms_played"`) even when names collide (content sniff
     precedence: Spotify keys win → not ours).
2. `parse(stream, ctx)` via `stream-json` streamArray. Per entry `e`:
   - **Music selection**: keep iff `e.header === 'YouTube Music'` OR
     (`typeof e.titleUrl === 'string'` AND its URL host is exactly `music.youtube.com`).
     Non-music → `skip:non_audio` — but do NOT count silently: aggregate count only (these
     may be millions of YouTube video rows).
   - **Ad exclusion**: `e.details` array containing any `{ name: 'From Google Ads' }` →
     `skip:ad`.
   - Title extraction FIRST (order is normative): from `e.title`, strip affixes via ordered
     rule list (`TITLE_AFFIX_RULES`): en prefix `/^Watched\s+/`; ja suffixes
     `/\s*を視聴しました$/` and `/\s*を再生しました$/`. No rule matched → the row still
     imports with the raw title as `candidateTitle`, counting
     `warning W_TITLE_AFFIX_UNKNOWN` once per file with 3 example titles.
   - **Deleted/unavailable** (checked on `candidateTitle` AFTER affix stripping):
     `candidateTitle` missing, matches `/^https?:\/\//`, or equals `e.titleUrl`; or no
     `titleUrl` AND no `subtitles` → `skip:deleted`.
   - `candidateTitle` empty after stripping → `skip:title_unparsed` (reserved exclusively
     for this case).
   - Artist extraction: `e.subtitles[0].name`, stripping trailing `/\s*-\s*Topic$/` and ja
     `/\s*- トピック$/`; missing subtitles → artist `Unknown Artist` + count
     `warning W_NO_ARTIST` (row imports; identity groups them under that artist).
   - Video ID: parse `titleUrl` with `new URL()`, take `v` query param when host is
     `music.youtube.com` or `www.youtube.com`; invalid URL → omit nativeIds.
   - Mapping: playedAt = `e.time` ISO → normalized `Z` (subsecond truncated); precision
     `'second'`; msPlayed omitted (ALWAYS — no estimates); playCount 1;
     nativeIds.youtubeVideoId; rawMeta: none in v1.
   - Entries that are not objects / missing `time` → `skip:invalid`.
   - Top-level non-array or malformed JSON → fatal `AppError('ERR_SCHEMA_MISMATCH')` with
     byte offset/path when available (same rule as issue 10).
   - T6 hardening identical to issue 10: depth > 20 → fatal `ERR_SCHEMA_MISMATCH`; string
     token > 64 KiB → `skip:invalid`; retained strings truncated at 1,000 chars with
     `W_FIELD_TRUNCATED`.
3. Fixtures: en export (music rows, plain YouTube rows, ad row, deleted row, no-subtitles
   row, `- Topic` and non-topic channels); ja export (`を視聴しました` titles,
   `- トピック`, localized folder name in zip); mixed zip with `watch-history.json` +
   `music-library-songs.csv` (ignored) + HTML history file (ignored).
4. Implement the optional `hintFor(entryName): string | undefined` parser method (contract
   already declared in issue 05's types; the issue 09 framework collects it into
   `ERR_NO_RECOGNIZED_FILES.details.hint`): returns `'youtube_html_format'` for basenames
   `watch-history.html` / `再生履歴.html`; undefined otherwise. The HTML-only-archive
   framework test lives here (test-level dependency on issue 09).

## Acceptance Criteria

- [ ] en/ja fixtures produce exact expected events; ja folder/file names match inside zip.
- [ ] Selection matrix: header-YTM w/ youtube.com URL (kept), header-YouTube w/ music URL
      (kept), header-YouTube w/ www URL (skipped), ad (skip:ad), deleted (skip:deleted).
- [ ] Affix behavior: unknown-locale affix → row imported with raw title + warning; stripped
      title empty → `skip:title_unparsed`.
- [ ] All events have `msPlayed === undefined` (asserted for every yielded event).
- [ ] HTML-only archive → `ERR_NO_RECOGNIZED_FILES` with `hint: 'youtube_html_format'`
      (framework test).
- [ ] Two entries with identical timestamp + track collapse into one event via the §6.5 hash
      (documented behavior); entries a few seconds apart import as two events (both tested).

## Validation

`npm test` (suite `src/importers/youtube.test.ts`), `npm run lint`, `npm run typecheck`.
Real ja-account export verification flagged as K1 checklist in PR.

## Dependencies

05-core-normalization (declares `hintFor` in the contract), 08-zip-ingestion (test-level),
09-import-framework (test-level, hint-surfacing test).

## Non-goals

HTML history parsing, Takeout music-library CSVs (v2), duration estimation (never),
persistence.

## Design References

DESIGN.md §6.4, §13.2 T6; docs/research/youtube-music-takeout.md (normative); ADR-002.
