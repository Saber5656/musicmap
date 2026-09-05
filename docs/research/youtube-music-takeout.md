# Research: YouTube Music History via Google Takeout

Status: verified via web sources on 2026-07-10. Entry-level details must be re-verified against a real export during implementation (see Open Questions).

## How users obtain it

1. Go to `takeout.google.com` and select only **YouTube and YouTube Music**.
2. Under "Multiple formats", change **History** from `HTML` (default) to **`JSON`** —
   this step is critical; the HTML format is not supported by musicmap v1.
3. Under "All YouTube data included", it is enough to select **history**.
4. Export is delivered as a ZIP within minutes to hours.
5. Relevant file: `Takeout/YouTube and YouTube Music/history/watch-history.json`
   (folder names are **localized** by account language, e.g. Japanese
   `Takeout/YouTube と YouTube Music/履歴/再生履歴.json`).

## `watch-history.json` structure

A single JSON array (can exceed 100 MB). Each entry:

```json
{
  "header": "YouTube Music",
  "title": "Watched <track title>",
  "titleUrl": "https://music.youtube.com/watch?v=<videoId>",
  "subtitles": [
    { "name": "<artist channel name> - Topic",
      "url": "https://www.youtube.com/channel/<channelId>" }
  ],
  "time": "2024-03-01T12:34:56.789Z",
  "products": ["YouTube"],
  "activityControls": ["YouTube watch history"]
}
```

| Field | Notes |
|---|---|
| `header` | `"YouTube Music"` for plays made in the YT Music app/site; `"YouTube"` otherwise. |
| `title` | `"Watched "` + title. **The prefix is localized** by account language (Japanese exports use a suffix `「…を視聴しました」`). Deleted/private videos appear as `"Watched https://..."`. |
| `titleUrl` | `music.youtube.com/watch?v=` host for YT Music plays; contains the video ID. May be absent for removed content. |
| `subtitles` | Channel of the content. Auto-generated artist channels end with `" - Topic"` — stripping that suffix yields the artist name. May be absent (deleted channels). |
| `time` | ISO 8601 UTC timestamp of the play event (start of playback). |
| `details` | When present with `{"name": "From Google Ads"}`, the entry is an **ad** and must be excluded. |

## Music-row selection rule

An entry is a music play if **either**:
- `header == "YouTube Music"`, or
- `titleUrl` host is `music.youtube.com`.

Both conditions must be checked: plays of music content in the regular YouTube app have
`header == "YouTube"` but a YT Music URL only when played via YT Music; v1 intentionally imports
only YT-Music-surface plays to avoid counting ordinary video watching as listening.

## Fidelity summary

| Property | Value |
|---|---|
| Coverage | As far back as YouTube history retention allows — users with auto-delete (3/18/36 months) have truncated history; noted as a coverage gap in UI |
| Granularity | Per play event (timestamp only) |
| Played duration | **No** — no `ms_played` equivalent at all |
| Track identity | Video ID (`v=` param) + title string |
| Album info | No |
| Genre info | No |
| Artist | Channel name heuristic (`" - Topic"` strip); imperfect for non-topic channels |
| Timezone | UTC |

## Implications for musicmap

- YTM events have `ms_played = null`. Duration-based metrics (minutes listened) must either
  exclude YTM or show count-based equivalents; the stats layer defines this policy explicitly
  per visualization. No silent fabrication of durations.
- Title prefix localization: v1 ships strip rules for English (`"Watched "` prefix) and Japanese
  (`"…を視聴しました"` / `"…を再生しました"` suffixes). When no rule matches, the row still
  imports with the raw title and the file gets a `W_TITLE_AFFIX_UNKNOWN` warning; the
  `title_unparsed` skip is reserved for titles that become empty after stripping (issue 12 is
  normative).
- Entries that are ads, deleted videos (URL-only titles), or missing both title and video ID are
  skipped and counted per-category in the import report.
- Consecutive duplicate events (same video ID within a small window) occur when playback is
  resumed; v1 keeps them (they are distinct play events) — documented behavior.
- The `Music library songs.csv` and playlist CSVs in Takeout are out of scope for v1.

## Open questions (verify with a real export during implementation)

1. Exact current localized folder/file names for en/ja accounts.
2. Whether `header` values are localized (reports suggest header stays `"YouTube Music"` /
   `"YouTube"` regardless of language — must confirm with a Japanese-language export).
3. Millisecond vs second precision of `time` in current exports.

## Sources

- https://data-goblins.com/power-bi/youtube-data-in-power-bi (structure, JSON vs HTML, limitations)
- https://github.com/seanbreckenridge/google_takeout_parser (parsing reference implementation)
- https://github.com/beebls/youtube-music-history-scrobbler (YT Music header/Topic-suffix handling)
- https://support.google.com/accounts/thread/191410174 (completeness caveats)
