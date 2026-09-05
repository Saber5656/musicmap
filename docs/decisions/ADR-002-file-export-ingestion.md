# ADR-002: File-export ingestion with a canonical play-event model

- Status: Accepted (2026-07-10)
- Deciders: product owner (Saber5656), Fable (design)

## Context

The product must reconstruct a full listening autobiography. Streaming APIs cannot do this:
Spotify's Web API returns only the ~50 most recent plays; Apple Music has no history API;
YouTube Music has no official API. All three vendors, however, provide full-history data
exports (GDPR-style). Sources differ wildly in fidelity: Spotify has per-ms played durations,
Apple has durations + partial genres + per-event UTC offsets, YouTube Music has timestamps only.

## Decision

1. v1 ingests **user-provided export files only** (Spotify Extended Streaming History ZIP/JSON,
   Apple Music Play Activity / Play History Daily Tracks CSV, Google Takeout
   `watch-history.json`). No OAuth, no vendor API calls for history.
2. All sources normalize into one **canonical play event**:
   `(source, played_at UTC, played_at_precision second|day, track, artist, ms_played nullable,
   play_count, raw_meta)` — nullable fidelity is explicit, never fabricated.
3. Missing data stays missing: duration-based metrics exclude events with `ms_played = NULL`
   (i.e. YouTube Music) and the UI labels this; day-precision events are excluded from
   time-of-day statistics.
4. Re-imports are **idempotent** via a content hash per event (`event_hash` unique index) and a
   per-upload SHA-256 duplicate check.
5. Raw uploaded files are retained on disk (local only) to allow future reprocessing when
   parsers improve; PII fields inside exports (IP addresses, device IDs, account IDs) are
   dropped at parse time and never enter the database.

## Consequences

- Deterministic, offline, testable imports; no token/refresh machinery.
- Users must perform a manual export request per service (days of latency) — acceptable for an
  autobiography tool; documented per-source in the import wizard.
- Cross-source metric comparability is imperfect by design; every visualization spec must state
  its policy for NULL durations and day precision (DESIGN.md §8, §11).
- Format drift of vendor exports is a permanent maintenance risk → parsers are alias-tolerant,
  fail with actionable schema-mismatch errors, and are covered by fixture tests per vintage.

## Alternatives considered

- **Spotify OAuth ingestion**: cannot recover history; only useful for metadata → deferred, and
  metadata comes from MusicBrainz instead (ADR-003).
- **Last.fm scrobble import**: valuable for scrobblers, but a 4th parser + API auth; deferred to v2.
- **Expanding daily aggregates into synthetic per-play events**: rejected — fabricated
  timestamps would corrupt time-of-day and streak statistics; `play_count` on one event row
  preserves totals honestly.
