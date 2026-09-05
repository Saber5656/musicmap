# Title

Enrichment job runner: FSM, persistence, status/control endpoints

## Summary

Implement the resumable artist-enrichment job per DESIGN.md §12.4: per-artist state rows,
most-listened-first work order, the idle/running/paused/completed FSM persisted in settings,
auto-pause on network failure streaks, genre upserts, and the
`/api/enrichment/status|start|pause` endpoints.

## Context

At 1 req/s a full first run takes an hour+ (research doc): restart-safety and honest progress
are the core requirements, not throughput. The runner composes issue 25's client and issue
24's consent gate.

## Scope

- `src/enrichment/runner.ts`, `src/db/queries/enrichment.ts`,
  `src/server/routes/enrichment.ts`; wiring into serve bootstrap + import-completion hook.

## Detailed Requirements

1. Query layer:
   - `seedPending(db)`: INSERT INTO artist_enrichment (artist_id, status='pending') for
     artists lacking a row (called on start and after each import completes).
   - `nextPending(db, limit)`: pending artists ordered by total plays DESC (join agg or
     events), id ASC tiebreak.
   - `recordMatch(db, artistId, mbid, score, genres: {name,count}[])`: transaction — upsert
     `genres` by lowercased name, replace `artist_genres` rows (votes = count), set row
     `matched` + fetched_at.
   - `recordUnmatched(db, artistId)`: sets status `unmatched` + `fetched_at` (no attempts
     increment — unmatched is a definitive answer, not a retriable failure).
   - `recordFailed(db, artistId)`: sets status `failed`, `attempts++`, `last_attempt_at`.
   - `counts(db)` → { pending, matched, unmatched, failed }.
2. Runner (`createEnrichmentRunner(deps)`; deps: db, mbClient, settings accessor, logger):
   - FSM per §12.4 diagram; state persisted to `settings['enrichment.jobState']` on every
     transition (values exactly: `idle|running|paused|completed`); in-memory singleton per
     process.
   - `start()`: requires `enrichment.enabled=true` (else `AppError('ERR_ENRICHMENT_DISABLED')`);
     seeds pending; state→running; loop: take next pending → searchArtist → pickArtistMatch →
     matched ? getArtistGenres + recordMatch : recordUnmatched. Item error mapping
     (exhaustive — the runner never crashes on upstream data, T13): `MbRequestError`
     (incl. zod/response-shape failures per issue 25) → `recordFailed`, continue;
     `MbUnavailableError` → `recordFailed(db, artistId)` AND consecutiveFailures++ — after
     10 consecutive, state→paused with `pauseReason='network'` BEFORE dispatching another
     item; any success resets the streak; anything else is a programmer bug (logged error,
     item recordFailed, streak++).
   - `pause()`: graceful — current item finishes, no new dispatch; state→paused
     (`pauseReason='user'`).
   - Settings hook (issue 24's `mmOnSettingChanged`): `enrichment.enabled=false` → pause
     (`pauseReason='disabled'`).
   - Completion: no pending artists left → state→completed (issue 27 later inserts an
     artwork phase before `completed`; design the loop so a second work-queue phase can be
     appended without FSM changes). Import-completed hook (transition order explicit):
     seedPending first; then, if the persisted state was `completed` AND enabled AND new
     pending rows exist → transition directly to `running` (auto-continue); in every other
     state (`idle`/`paused`) stay put — the user starts/resumes manually.
   - Boot recovery: persisted `running` at process start → set `paused`
     (`pauseReason='restart'`) — user resumes explicitly (no surprise network on boot).
   - Retry semantics for `failed` items: `start()` re-seeds items with status='failed' AND
     attempts < 3 back to pending (bounded auto-retry per repo policy).
3. Routes (§9):
   - `GET /api/enrichment/status` → `{ state, phase: 'artist'|'artwork', pauseReason?,
     counts, artworkCounts?, currentArtist?: { id, name }, etaSeconds?, lastError? }`
     (eta = pending × 1.2 s, null unless running; `lastError` = short string for the most
     recent pause/error cause, e.g. `'network'`, else omitted — §12.4). In THIS issue
     `phase` is always `'artist'` and `artworkCounts` is omitted — issue 27 adds the
     artwork phase behind this already-shipped contract (§12.4/§12.5).
   - `POST /api/enrichment/start` → 202 `{ state }`; 409 `ERR_ENRICHMENT_DISABLED` when
     consent off; idempotent when already running (202, no-op).
   - `POST /api/enrichment/pause` → 202; idempotent.
4. Observability: log line per 50 items (counts snapshot); NO artist names at info level
   (privacy of logs — T14; debug level may include them).

## Acceptance Criteria

- [ ] Mocked-client integration test: 20-artist library → matched/unmatched/failed land per
      mock script; genres upserted with votes; most-played-first order asserted.
- [ ] FSM transitions all tested: start-disabled 409; user pause mid-run; disable-setting
      pause; 10× MbUnavailable → network pause (fake timers); resume continues from
      persisted rows (fresh runner instance, same DB — restart simulation); completion; new
      import → completed+enabled auto-restarts, idle does not.
- [ ] Boot recovery: persisted `running` becomes `paused/restart` on runner construction.
- [ ] failed retry bound: attempts reach 3 → start() no longer re-seeds them.
- [ ] Status payload shape (zod contract test); counts match DB truth mid-run.
- [ ] Route security: status/start/pause each 401 without bearer token (registered inside
      the issue-06 hook chain).
- [ ] Network-pause path: each MbUnavailable item is recorded `failed` (no item left
      `pending` behind the cursor) and the pause happens before the 11th dispatch.
- [ ] Rate: with fake timers, ≥1,100 ms between MB dispatches through the whole runner path.
- [ ] Genre re-run idempotency: enriching the same artist twice leaves one genre row set
      (replace semantics).

## Validation

`npm test` (suite `src/enrichment/runner.test.ts` — zero real network), lint, typecheck.
Manual: against real MusicBrainz with a 5-artist toy library, paste status JSON before/after
(1 run, respectful of rate limits).

## Dependencies

24-settings-and-consent, 25-musicbrainz-client.

## Non-goals

Artwork (27), genre-map computation (28), track-level enrichment (v2), UI beyond what 24
already ships (map page shows progress — 28).

## Design References

DESIGN.md §12.4 (normative FSM), §12.2 (consent gate), §9 (endpoints), §13.2 T14;
ADR-003 (retry/order policy).
