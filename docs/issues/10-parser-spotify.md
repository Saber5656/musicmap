# Title

Spotify Extended Streaming History parser

## Summary

Implement the `spotify` ImportSourceParser: stream-parse `Streaming_History_Audio_*.json`
files (inside the export ZIP or uploaded loose), map fields to `CanonicalPlayEvent`, exclude
podcasts/audiobooks/video, drop PII, and tolerate known format drift — per
`docs/research/spotify-extended-streaming-history.md` and DESIGN.md §6.4.

## Context

Spotify is the highest-fidelity source (per-ms durations, track URIs, full lifetime). The
parser is a pure per-file stream; persistence/reporting is the framework's job (issue 09).

## Scope

- `src/importers/spotify.ts` + fixtures `fixtures/exports/spotify/` + tests.
- Add dependency `stream-json`.

## Detailed Requirements

1. `matches(entryName, peek)` — evaluation order is normative:
   1. If basename matches `/^Streaming_History_Video/i` → return false (before any sniff).
   2. If basename matches `/^Streaming_History_Audio.*\.json$/i` → return true.
   3. Otherwise, any `.json` whose peek content-sniffs to a Spotify event array: peek
      (first 4 KiB) trimmed starts with `[` and contains BOTH `"ts"` and `"ms_played"`.
   - Must reject `watch-history.json` and CSV peeks (negative fixtures).
2. `parse(stream, ctx)` uses `stream-json` (`parser` + `streamArray`) — never buffers the
   whole file. Skip notation used below: `skip:podcast` means emitting
   `{ kind: 'skip', reason: 'podcast' }` (same shorthand for every reason). For each
   element `r`:
   - Podcast/audiobook/video row: `spotify_track_uri` null/absent OR `episode_name`/
     `audiobook_title` present → `skip:podcast` (use `skip:audiobook` when audiobook fields
     present); rows with all metadata null AND `ms_played === 0` → `skip:invalid`.
   - T6 hardening (DESIGN §13.2): JSON nesting depth > 20 inside an element → fatal
     `ERR_SCHEMA_MISMATCH`; any incoming string value > 64 KiB → `skip:invalid` for that row
     (structurally required fields) — the 1,000-char truncation below applies only to
     retained values under that ceiling.
   - Field mapping. NO aliases are required for the fields below — exact names only; if a
     real export deviates (format drift, K1), update `docs/research/` + fixtures + this
     table rather than guessing. Unknown extra fields are ignored but reported once per file
     as `warning W_UNKNOWN_FIELDS` listing up to 10 names:
     | Canonical | Source field | Rule |
     |---|---|---|
     | playedAt | `ts` | must parse ISO 8601; output normalized `YYYY-MM-DDTHH:MM:SSZ`; invalid → `skip:invalid` |
     | precision | — | always `'second'` |
     | artistName | `master_metadata_album_artist_name` | trim; empty/null → `skip:invalid` |
     | trackTitle | `master_metadata_track_name` | trim; empty/null → `skip:invalid` |
     | album | `master_metadata_album_album_name` | trim; empty → omit |
     | msPlayed | `ms_played` | integer ≥ 0; negative → 0 with `warning W_NEGATIVE_MS` (count once per file); non-number → `skip:invalid` |
     | playCount | — | always 1 |
     | nativeIds.spotifyTrackUri | `spotify_track_uri` | keep verbatim when matching `/^spotify:track:[A-Za-z0-9]+$/`, else omit |
     | rawMeta.reason_end | `reason_end` | string, ≤ 64 chars, else truncate |
     | rawMeta.platform | `platform` | string, ≤ 64 chars, else truncate + `W_FIELD_TRUNCATED` |
     | rawMeta.shuffle / skipped / incognito_mode | same names | booleans only |
   - **PII rule (T10)**: `ip_addr`, `ip_addr_decrypted`, `user_agent_decrypted`, `username`,
     `conn_country`, `offline_timestamp` are NEVER copied to the event (explicit denylist —
     copy nothing outside the table above).
   - String fields capped at 1,000 chars (longer → truncate + `warning W_FIELD_TRUNCATED`).
   - Malformed JSON mid-file → throw `AppError('ERR_SCHEMA_MISMATCH')` with byte offset in
     details (fatal — the file is unusable, framework fails the import).
   - Top-level not-an-array → `ERR_SCHEMA_MISMATCH`.
3. Fixtures (small, hand-written, committed):
   `audio_2019.json` (8 music rows covering all mapping cases incl. unicode ja titles),
   `audio_with_podcasts.json`, `audio_older_vintage.json` (has `username`, `ip_addr_decrypted`),
   `audio_newer_vintage.json` (has `audiobook_*`, coarse `platform`), `broken.json`
   (truncated), `video_history.json` (must not match), plus a mini export ZIP assembled by the
   fixture script containing 2 audio files + video file + PDF.

## Acceptance Criteria

- [ ] All fixtures parse to the exact expected `CanonicalPlayEvent[]`/skip counts (snapshot or
      literal expectations in test).
- [ ] PII fields provably absent from every yielded event (test iterates all keys, asserts
      only whitelisted rawMeta keys occur).
- [ ] ZIP fixture end-to-end through issue 09 framework: only audio files matched; PDF/video
      untouched; correct totals.
- [ ] Memory: parsing a generated 100 MB JSON (fixture script, gitignored) keeps RSS growth
      < 200 MiB (test with `--expose-gc` heuristic or skip in CI; document command).
- [ ] `matches` negative cases (YTM json, Apple csv peek) pass.

## Validation

`npm test` (suite `src/importers/spotify.test.ts`), lint, typecheck. Update
`docs/research/spotify-extended-streaming-history.md` "Open questions" with anything learned
from real data if available (do not block on it).

## Dependencies

05-core-normalization (types), 08-zip-ingestion (test-level), 09-import-framework
(test-level, for the ZIP end-to-end acceptance criterion).

## Non-goals

Basic "Account data" `StreamingHistory_music_*.json` (v2 — DESIGN §2.3), video/podcast
analytics, persistence (09).

## Design References

DESIGN.md §6.4 (contract, common rules), §13.2 T6/T10;
docs/research/spotify-extended-streaming-history.md (normative field semantics); ADR-002.
