# Research: Spotify Extended Streaming History Export

Status: verified via web sources on 2026-07-10. Field-level details must be re-verified against a real export during implementation (see Open Questions).

## How users obtain it

1. Go to Spotify **Account > Privacy settings** (`spotify.com/account/privacy`).
2. Request **Extended streaming history** (this is a separate checkbox from the basic "Account data" package).
3. Spotify emails a download link. Delivery takes from a few days up to 30 days.
4. The download is a ZIP archive.

## Archive layout

- ZIP contains a folder such as `Spotify Extended Streaming History/`.
- Playback data is split across multiple JSON files, commonly named
  `Streaming_History_Audio_<year-range>_<n>.json` (music/podcasts) and optionally
  `Streaming_History_Video_<...>.json`.
- Each JSON file is a single JSON **array** of event objects (typically ~10-20 MB per file).
- A PDF "Read Me First" document is included.

The parser MUST select files by content shape (array of objects with `ts` and `ms_played`), not
only by exact filename, because naming has changed across export vintages.

## Event object fields

| Field | Type | Semantics / notes |
|---|---|---|
| `ts` | string, ISO 8601 UTC | Timestamp of when the stream **ended**. Second precision. |
| `username` | string | Present in some export vintages; treat as optional. |
| `platform` | string | Device/OS. Coarse values (`android`, `windows`, ...) after 2023-10-19; older exports have detailed strings. |
| `ms_played` | integer | Milliseconds actually played. |
| `conn_country` | string | ISO country code of connection. |
| `ip_addr` / `ip_addr_decrypted` | string | IP address. **PII — must be dropped at parse time, never stored.** Name varies by vintage. |
| `user_agent_decrypted` | string/null | Present in some vintages. **PII — drop.** |
| `master_metadata_track_name` | string/null | Track title. `null` for podcast/audiobook rows. |
| `master_metadata_album_artist_name` | string/null | Artist name. |
| `master_metadata_album_album_name` | string/null | Album title. |
| `spotify_track_uri` | string/null | `spotify:track:<base62id>`. Non-null identifies a **music** row. |
| `episode_name` | string/null | Podcast episode title. |
| `episode_show_name` | string/null | Podcast show title. |
| `spotify_episode_uri` | string/null | Podcast episode URI. |
| `audiobook_title`, `audiobook_uri`, `audiobook_chapter_uri`, `audiobook_chapter_title` | string/null | Present in newer vintages only. Parser must tolerate absence/presence. |
| `reason_start` | string | Enum-ish: `appload`, `playbtn`, `trackdone`, `fwdbtn`, `backbtn`, `remote`, ... New values appear over time; treat as open enum. |
| `reason_end` | string | Same open-enum caveat (`trackdone`, `endplay`, `fwdbtn`, `logout`, `unexpected-exit-while-paused`, ...). |
| `shuffle` | boolean | Shuffle on/off. |
| `skipped` | boolean/null | Unreliable before 2022-10-16. |
| `offline` | boolean/null | Offline playback. |
| `offline_timestamp` | integer/null | Historically inconsistent units (seconds vs milliseconds). Do not rely on it. |
| `incognito_mode` | boolean/null | Private session. |

## Fidelity summary

| Property | Value |
|---|---|
| Coverage | Full account lifetime |
| Granularity | Per-stream event |
| Played duration | Yes (`ms_played`, ms precision) |
| Track identity | Strong (`spotify_track_uri`) |
| Album info | Yes |
| Genre info | **No** |
| Timezone | Timestamps are UTC |

## Implications for musicmap

- Music rows are selected by `spotify_track_uri != null`. Podcast/audiobook rows are excluded in v1.
- `ip_addr*`, `user_agent*`, `username` are PII: the parser drops them; they never reach the database.
  The retained raw upload still contains them, so the raw-file store must be treated as sensitive
  (local only, documented).
- Very short plays (`ms_played` below a threshold) should still be imported; filtering is a
  stats-layer policy, not a parse-time decision.
- Format drift across vintages is real (community reports of renamed tags). The parser needs
  per-field tolerant mapping with alias support and an "unknown field" pass-through policy.
- The basic (non-extended) "Account data" package contains only
  `StreamingHistory_music_*.json` with `{endTime, artistName, trackName, msPlayed}` limited to
  the last year. **v1 does not support it** (documented limitation; candidate for v2).

## Open questions (verify with a real export during implementation)

1. Exact current file naming and folder name in a 2026 export.
2. Whether `username` / `ip_addr` field names appear in current vintage and under which names.
3. Presence and shape of `audiobook_*` fields.

## Sources

- https://blog.ortham.net/posts/2024-12-21-spotify-streaming-history-part-1/ (field-by-field analysis of a real export)
- https://ericchiang.github.io/post/spotify/
- https://github.com/Yooooomi/your_spotify/issues/458 (format-drift evidence)
- https://community.spotify.com/t5/Other-Podcasts-Partners-etc/Extended-Streaming-History-missing-track-metadata/td-p/6269152
