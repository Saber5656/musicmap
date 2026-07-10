# Research: Metadata Enrichment APIs (genre, artwork)

Status: verified via web sources on 2026-07-10.

## Why enrichment is needed

None of the three supported exports carries a usable genre taxonomy (Apple's `Genre` column is
partial and single-label). The genre map and genre-mode trends require artist→genre data from an
external source. Per the frozen requirements, enrichment is **opt-in**, sends only artist/track
names (never play history), and results are cached locally.

## Candidate APIs

### MusicBrainz Web Service v2 — SELECTED for v1

| Property | Value |
|---|---|
| Base URL | `https://musicbrainz.org/ws/2/` (append `fmt=json`) |
| Auth | **None required** for read access |
| Rate limit | **1 request/second per IP** (average); exceeding it can lead to IP blocks. Global anonymous limits also apply. |
| Required header | Meaningful `User-Agent` identifying app, version, and contact, e.g. `musicmap/1.0.0 (+https://github.com/Saber5656/musicmap)` — requests without it may be rejected |
| Throttle response | `503` (respect and back off) |
| Data license | Core data is CC0 |

Endpoints used by musicmap v1:

| Purpose | Endpoint |
|---|---|
| Resolve artist name → MBID | `GET /ws/2/artist?query=artist:"<name>"&fmt=json&limit=5` (returns match `score` 0-100) |
| Artist genres | `GET /ws/2/artist/<mbid>?inc=genres&fmt=json` (use `genres`, not `genre-rels`) |
| Resolve recording (optional, v2) | `GET /ws/2/recording?query=...` |
| Release-group for artwork | `GET /ws/2/release-group?query=releasegroup:"<album>" AND artist:"<artist>"&fmt=json` |

Matching policy (design-level): accept the top search hit when `score >= 90` and it is at least
5 points above the second hit; otherwise mark the artist `unmatched` (re-triable). No fuzzy
auto-accept below threshold — false genre attribution is worse than a gap.

### Cover Art Archive — SELECTED for v1 (artwork)

| Property | Value |
|---|---|
| Base URL | `https://coverartarchive.org/` |
| Auth | None |
| Endpoint | `GET /release-group/<mbid>/front-500` → `307` redirect to the image on archive.org; `404` when no image chosen |
| Sizes | 250 / 500 / 1200 thumbnails |

Images are cached to local disk after first fetch; the UI never hotlinks.

### ListenBrainz metadata lookup — REJECTED for v1

`GET/POST /1/metadata/lookup` (bulk artist_name+recording_name → MBID mapping) would be ideal for
batch resolution, **but the endpoint now requires an auth token** (anti-scraper change) and
per-user tokens fall under the "user-provided API keys" capability that the frozen requirements
defer to v2. Rate limiting is dynamic via `X-RateLimit-*` headers.

### Last.fm / Spotify Web API — REJECTED for v1

Both require user-provisioned API keys/OAuth apps → deferred to v2 by requirements.

## Privacy posture

- Enrichment runs only after explicit opt-in in Settings (default OFF).
- Outbound payload is limited to artist names / album titles (and app User-Agent). Play
  timestamps, counts, and any history-derived data are never sent.
- All outbound requests go to two hosts only: `musicbrainz.org`, `coverartarchive.org`
  (+ the archive.org redirect target for images). This allowlist is enforced in code and stated
  in SECURITY docs.
- Responses are cached in SQLite with a fetch timestamp; re-fetch only on explicit user action
  ("refresh enrichment") or cache age > 90 days.

## Throughput reality check

At 1 req/s: an average library of ~2,000 distinct artists needs ~2,000 search calls +
~N lookup calls for matched artists ≈ 60-90 minutes of background enrichment on first run.
The job runner must therefore be resumable, persist per-artist state, and survive app restarts.
Artwork fetches (top albums only, bounded, e.g. 200 release-groups) add a few minutes.

## Sources

- https://musicbrainz.org/doc/MusicBrainz_API (endpoints, User-Agent requirement)
- https://musicbrainz.org/doc/MusicBrainz_API/Rate_Limiting (1 req/s per IP; block policy)
- https://musicbrainz.org/doc/Cover_Art_Archive/API (front-cover endpoint, 307/404, sizes)
- https://listenbrainz.readthedocs.io/en/latest/users/api/metadata.html (lookup endpoint; token now required)
- https://listenbrainz.readthedocs.io/en/latest/users/api/index.html (rate-limit headers)
