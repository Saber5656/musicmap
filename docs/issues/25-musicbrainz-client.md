# Title

MusicBrainz and Cover Art Archive HTTP clients with rate limiting and host allowlist

## Summary

Implement the outbound HTTP layer per DESIGN.md §12.3/§12.5 and ADR-003: a single allowlisted
fetch wrapper, a 1-req/1100ms token-bucket MusicBrainz client (artist search, artist+genres
lookup, release-group search) with backoff and zod-validated responses, and the CAA
front-cover fetcher with redirect re-validation and image caps.

## Context

This is the ONLY module that talks to the internet. Threats T9/T13 are enforced here once, so
the runner (26) and artwork (27) inherit them. Everything must be testable offline via undici
MockAgent.

## Scope

- `src/enrichment/http.ts` (allowlist wrapper), `src/enrichment/musicbrainz.ts`,
  `src/enrichment/coverart.ts`, rate limiter util `src/enrichment/rateLimiter.ts`.
- devDep `undici` (MockAgent) for tests.

## Detailed Requirements

1. `http.ts`:
   ```ts
   allowedHost(url: URL): boolean
   // EXACT matches only: host === 'musicbrainz.org' || host === 'coverartarchive.org'
   //   || host === 'archive.org' || host.endsWith('.archive.org')
   // (subdomain matching for archive.org ONLY — 'evil.musicbrainz.org' must be rejected;
   //  archive.org hosts are legitimate solely as CAA redirect targets)
   safeFetch(url, init, { maxRedirects = 3 }): Promise<Response>
   ```
   - `safeFetch` is exported as an enrichment-internal helper used only by
     `musicbrainz.ts`, `coverart.ts`, and tests; no module in `src/enrichment` may call
     global `fetch` directly (lint-guard note).
   - Rejects non-https URLs and disallowed hosts with `AppError('ERR_FORBIDDEN_HOST')`
     BEFORE dispatching.
   - Manual redirect mode: `redirect: 'manual'`; each 3xx Location re-validated with
     `allowedHost` (absolute or relative resolution) — violation → `ERR_FORBIDDEN_HOST`;
     exceeding maxRedirects → `MbUnavailableError` on the MB path, `{ status: 'failed',
     reason: 'redirect_limit' }` on the CAA path.
   - Sets `User-Agent: musicmap/<version> (+https://github.com/Saber5656/musicmap)` on every
     request; version injected, not read from disk here.
   - Per-request timeout 30 s via `AbortSignal.timeout`.
2. `rateLimiter.ts`: `createLimiter(minIntervalMs)` → `schedule<T>(fn): Promise<T>` FIFO
   queue guaranteeing ≥ minInterval between dispatch times (not completions); injectable
   clock for fake-timer tests.
3. Test seam (owned HERE because this module is the single outbound boundary; issue 30
   consumes it): env var `MUSICMAP_ENRICHMENT_BASE_URL_OVERRIDE` — a JSON map of
   allowlisted domain → local base URL (e.g. `{"musicbrainz.org":
   "http://127.0.0.1:<mockPort>/mb"}`). Semantics: activates ONLY when
   `NODE_ENV === 'test'` (inert in production — unit-tested); when active, `safeFetch`
   rewrites mapped hosts BEFORE dispatch and REJECTS (`ERR_FORBIDDEN_HOST`) any allowlisted
   host that has no mapping — so a mis-mocked E2E test fails instead of silently reaching
   the real service.
4. MusicBrainz client (all through ONE limiter instance at 1,100 ms; JSON via `fmt=json`):
   - `searchArtist(name)` → `GET /ws/2/artist?query=artist:"<escaped>"&fmt=json&limit=5`;
     Lucene escaping: quote the name, escape `"` and `\` inside; returns
     `{ artists: [{ id, name, score }] }` (zod; extra fields stripped).
   - `getArtistGenres(mbid)` → `GET /ws/2/artist/<mbid>?inc=genres&fmt=json` → `{ genres:
     [{ name, count }] }` (absent array → []).
   - `searchReleaseGroup(artistName, album)` → query
     `releasegroup:"<album>" AND artist:"<artist>"`, limit 3 → `[{ id, score, title }]`.
   - Match policy helper `pickArtistMatch(results)` per ADR-003: top score ≥ 90 AND
     (single result OR lead ≥ 5 over #2) → `{ mbid, score }`; else null.
   - Retry/backoff per §12.3: on 503/429 → wait `Retry-After` (seconds header, capped 60 s)
     else 2 s/8 s/30 s ladder — max 3 retries, i.e. 4 total attempts — still failing →
     throw `MbUnavailableError` (typed) so the runner can count consecutive failures; on
     4xx (except 429) → typed `MbRequestError` (item-level failure, no retry);
     network/timeout errors → same ladder. Zod/JSON response-shape failures also classify
     as `MbRequestError` (item-level, no retry) — the runner must never crash on malformed
     upstream data (T13).
5. CAA client (own limiter, 500 ms):
   ```ts
   fetchFrontCover(releaseGroupMbid: string, sink: NodeJS.WritableStream): Promise<
     | { status: 'fetched'; contentType: 'image/jpeg' | 'image/png'; bytes: number }
     | { status: 'not_found' }
     | { status: 'failed'; reason: 'too_large' | 'bad_content_type' | 'write_failed'
                          | 'redirect_limit' | 'transport' }>
   ```
   - GET `https://coverartarchive.org/release-group/<mbid>/front-500` following redirects
     via safeFetch; 404 → `not_found`; 200 → validate `content-type` BEFORE writing any
     byte to `sink` (`bad_content_type` otherwise); stream with a 2 MiB hard cap (abort
     beyond → `too_large`); sink write errors → `write_failed`; network/timeout →
     `transport`. The caller (issue 27) owns temp-file + rename semantics.
6. NO database access in this issue's modules (pure HTTP layer). `safeFetch` is exported
   only for use by the enrichment clients and tests (per item 1); global `fetch` is never
   called directly anywhere in `src/enrichment`.

## Acceptance Criteria

- [ ] MockAgent tests: search/lookup/release-group happy paths parse; Lucene escaping of
      `Weird "Al" \ Yankovic` produces the exact expected query string.
- [ ] Rate limiter: 5 scheduled calls with fake timers dispatch at t=0,1100,2200,... (MB) —
      asserted; CAA at 500 ms independent.
- [ ] Backoff: 503 with Retry-After: 3 → next attempt at +3000 ms; FOUR consecutive 503
      responses (initial + 3 retries) → MbUnavailable; 404 artist lookup → MbRequestError
      (no retries — call count asserted); malformed MB JSON body → MbRequestError; CAA
      wrong content-type → `bad_content_type` with zero bytes written to the sink.
- [ ] Allowlist: `http://musicbrainz.org` (non-https), `https://evil.com`,
      `https://musicbrainz.org.evil.com` all rejected pre-dispatch; redirect to
      `https://ia800000.us.archive.org/...` allowed; redirect to non-allowlisted host
      rejected (MockAgent 307 chains).
- [ ] Cover cap: 3 MiB mock body aborts at 2 MiB with `too_large`.
- [ ] Every outbound request in every test carries the exact User-Agent (interceptor assert).
- [ ] `pickArtistMatch` table-tested: (95), (95,91), (95,89), (89), (90 single) → expected.
- [ ] Test seam: with `NODE_ENV=test` + override map, mapped hosts rewrite to the local
      base URL and unmapped allowlisted hosts are rejected; with `NODE_ENV=production` the
      override env var is completely ignored (both asserted).

## Validation

`npm test` (suite `src/enrichment/*.test.ts`, zero real network — enforced by MockAgent
`disableNetConnect`), lint, typecheck.

## Dependencies

01-project-scaffold, 03-config-and-paths (version/UA injection pattern only).

## Non-goals

Job orchestration/persistence (26), artwork DB rows/files (27), ListenBrainz/Last.fm (v2).

## Design References

DESIGN.md §12.3, §12.5, §13.2 T9/T13; ADR-003 (match policy, allowlist);
docs/research/metadata-enrichment-apis.md.
