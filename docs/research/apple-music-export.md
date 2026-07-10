# Research: Apple Music Data Export (privacy.apple.com)

Status: verified via web sources on 2026-07-10. Column-level details must be re-verified against a real export during implementation (see Open Questions).

## How users obtain it

1. Sign in at `privacy.apple.com` > **Get a copy of your data**.
2. Request **Apple Media Services information** (App Store, iTunes Store, Apple Music activity).
3. Apple prepares a ZIP archive, typically within 1-7 days.
4. Relevant files live under a path like
   `Apple Media Services information/Apple_Media_Services/Apple Music Activity/`.

## Relevant files

| File | Granularity | Use in musicmap |
|---|---|---|
| `Apple Music Play Activity.csv` | Per playback event | **Primary source** |
| `Apple Music - Play History Daily Tracks.csv` | Per track per day (aggregated) | **Fallback** when Play Activity is absent |
| `Apple Music - Recently Played Tracks.csv` | Recent items only | Not used (v1) |
| `Apple Music Library Tracks.json/csv` | Library metadata | Not used (v1) |
| `Identifier Information.json/csv` | Track title ↔ internal ID | Not used (v1) |

File naming and the exact folder nesting vary across export vintages; the importer must locate
files by name pattern anywhere inside the archive.

## `Apple Music Play Activity.csv` — columns

Reported columns (union across vintages; **order and presence vary**):

`Apple Id Number`, `Apple Music Subscription`, `Artist Name`, `Build Version`,
`Client IP Address`, `Content Name` (a.k.a. `Song Name` in some vintages), `Content Provider`,
`Content Specific Type`, `Device Identifier`, `End Position In Milliseconds`,
`End Reason Type`, `Event End Timestamp`, `Event Reason Hint Type`, `Event Received Timestamp`,
`Event Start Timestamp`, `Event Type`, `Feature Name`, `Genre`, `Item Type`,
`Media Duration In Milliseconds`, `Media Type`, `Metrics Bucket Id`, `Metrics Client Id`,
`Milliseconds Since Play`, `Offline`, `Original Title`, `Play Duration Milliseconds`,
`Provider`, `Source Type`, `Start Position In Milliseconds`, `Store Country Name`,
`UTC Offset In Seconds`

Key mappings:

| Concept | Column(s) |
|---|---|
| Track title | `Content Name` (alias: `Song Name`) |
| Artist | `Artist Name` |
| Album | **Not present** in Play Activity |
| Genre | `Genre` (Apple's own labels — partial genre data, unlike Spotify/YTM exports) |
| Played ms | `Play Duration Milliseconds` |
| Event start | `Event Start Timestamp` (ISO 8601, UTC) |
| Event end | `Event End Timestamp` |
| Local timezone | `UTC Offset In Seconds` |
| Event kind | `Event Type` (e.g. `PLAY_END`, `LYRIC_DISPLAY`, ...) |
| Media kind | `Media Type` (`AUDIO` vs video) |

## `Apple Music - Play History Daily Tracks.csv` — columns

Reported columns include: `Country`, `Track Identifier`, `Media type`, `Date Played`
(`YYYYMMDD`), `Hours`, `Play Duration Milliseconds`, `End Reason Type`, `Source Type`,
`Play Count`, `Skip Count`, `Ignore For Recommendations`, `Track Reference`,
`Track Description` (combined `"Artist - Title"` string that must be split on the first `" - "`).

Day-level granularity only; musicmap reconstructs timestamps as `YYYY-MM-DDT00:00:00Z` with a
`precision=day` marker (DESIGN.md §5 is normative; day-precision rows are excluded from
time-of-day statistics, so no local-noon fabrication is needed).

## Known quirks

- **Header drift across vintages** (e.g. `Song Name` vs `Content Name`); importer needs a
  column-alias map and must fail with a clear error listing found vs expected headers.
- Rows with blank `Event Start Timestamp` but present `Event End Timestamp` (and vice versa):
  derive the missing one using `Play Duration Milliseconds` when possible.
- `Play Duration Milliseconds` outliers: negative values and values far exceeding
  `Media Duration In Milliseconds` occur; clamp to `[0, Media Duration]` when media duration is
  present, else to a configurable ceiling.
- Non-playback rows (`Event Type` = `LYRIC_DISPLAY`, etc.) must be filtered out; only rows that
  represent completed/partial playback (`PLAY_END`, and `Media Type = AUDIO`) are imported.
- CSV may include a UTF-8 BOM; fields are quoted; embedded commas/newlines occur in titles.
- **PII columns** (`Apple Id Number`, `Client IP Address`, `Device Identifier`,
  `Metrics Client Id`) must be dropped at parse time and never stored.

## Fidelity summary

| Property | Play Activity | Daily Tracks |
|---|---|---|
| Coverage | Full subscription lifetime | Full |
| Granularity | Per event | Per track per day |
| Played duration | Yes (ms) | Yes (ms, aggregated) |
| Track identity | Title + artist strings only | Combined description string |
| Album info | No | No |
| Genre info | Yes (Apple labels) | No |
| Timezone | UTC + `UTC Offset In Seconds` | Date only |

## Open questions (verify with a real export during implementation)

1. Current exact filenames/paths in a 2026 export (Apple has reorganized folders before).
2. Whether `Content Name` or `Song Name` is the current header, and full current header list.
3. `Event Type` value inventory in current exports; confirm the correct playback filter.
4. Whether Play Activity is still included for all regions/accounts (some users reportedly only
   receive Daily Tracks — hence the fallback parser).

## Sources

- https://medium.com/swlh/apple-music-activity-analyser-part-1-dd02173f095f (column analysis of a real export)
- https://github.com/nerveband/Apple-Music-Play-History-Converter (supports Play Activity / Recently Played / Daily Tracks; evidence of header variants)
- https://mystats.music/guides/apple-music-history (how-to-obtain steps)
- https://discussions.apple.com/thread/251189905
