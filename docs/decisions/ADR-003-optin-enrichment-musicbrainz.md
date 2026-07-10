# ADR-003: Opt-in metadata enrichment via keyless MusicBrainz + Cover Art Archive

- Status: Accepted (2026-07-10)
- Deciders: product owner (Saber5656) — opt-in posture; Fable (design) — provider selection

## Context

No export format carries a usable genre taxonomy, and only artwork-less metadata is available
offline, but the genre map and genre trends need artist→genre data. The user decided:
enrichment must be **opt-in**, must send only artist/track names (never play history), and v1
must not require user-provisioned API keys.

## Decision

- v1 enrichment uses exactly two external services, both keyless:
  - **MusicBrainz WS/2** for artist-name→MBID resolution and artist genres
    (`inc=genres`), respecting the 1 request/second/IP limit with a descriptive `User-Agent`.
  - **Cover Art Archive** for release-group front covers (307-redirect flow), cached to disk.
- Enrichment is OFF by default; enabling it shows exactly what data leaves the machine.
- Outbound HTTP is restricted by a code-level host allowlist:
  `musicbrainz.org`, `coverartarchive.org`, `*.archive.org` (redirect targets). Anything else is
  a bug and a test failure.
- Artist matching: accept top search hit only when `score >= 90` **and** lead over the runner-up
  `>= 5`; otherwise `unmatched` (no fuzzy auto-accept — wrong genres are worse than gaps).
- All responses cached in SQLite (and images on disk); re-fetch only on user action or cache age
  > 90 days.
- **ListenBrainz `metadata/lookup` is rejected for v1**: the endpoint now requires an auth
  token, which collides with the "no user API keys in v1" requirement. Last.fm / Spotify Web API
  enrichment likewise deferred to v2.

## Consequences

- First full enrichment of ~2,000 artists takes ~60-90 minutes at 1 req/s → the job runner must
  be persistent, resumable, and pausable (DESIGN.md §12), with visible progress.
- Genre vocabulary = MusicBrainz community genres; some artists will be unmatched or genreless —
  the genre map must present an explicit "unknown" share rather than hiding it.
- Zero secrets in v1: no key storage, no OAuth flows, smaller threat surface.

## Alternatives considered

- **ListenBrainz bulk lookup**: ideal batching, but token-gated now → v2 (optional token setting).
- **Bundled static genre dataset**: no network at all, but licensing is murky (musicmap.info is
  proprietary; Every Noise data is Spotify-derived) and staleness is guaranteed; rejected.
- **Last.fm tags**: rich folksonomy but requires an API key; v2.
