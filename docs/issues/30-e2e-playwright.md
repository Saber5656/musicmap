# Title

End-to-end Playwright journey tests with MusicBrainz mock

## Summary

Implement the full-product E2E suite per DESIGN.md §16: boot the real built app against a
temp data dir, drive the complete user journey (import all 3 sources → all 4 visualizations →
chapters CRUD → settings/consent → mocked enrichment → genre map), plus an axe-core
accessibility smoke per page, wired into CI.

## Context

This is the whole-product validation gate (ISSUE_PLAN §6.3). External network is forbidden in
CI: MusicBrainz/CAA are mocked by a local HTTP server, reached through the
`MUSICMAP_ENRICHMENT_BASE_URL_OVERRIDE` test seam that issue 25 implements and unit-tests
(active only under `NODE_ENV=test`; unmapped allowlisted hosts are REJECTED while the
override is active, so any request that would have escaped to a real service fails the test
instead — this is the egress guard).

## Scope

- `e2e/` specs + helpers; `playwright.config.ts` finalized; mock server
  `e2e/support/mockMetabrainz.ts`; CI job addition to `.github/workflows/ci.yml`.

## Detailed Requirements

1. Harness: global setup builds once (`npm run build`), generates `small` fixtures (issue
   29), starts `musicmap serve` (spawned, temp `MUSICMAP_DATA_DIR`, `--no-open`,
   `--port 14949` — fail fast if unavailable) and captures the tokenized URL from stdout.
   Token hygiene (T11): exactly ONE untraced handshake smoke test navigates via the URL
   fragment like a real user; all other specs seed `sessionStorage['mm.token']` directly in
   the browser context, and a global teardown assertion greps every produced artifact
   (traces, screenshots, reports) for the token substring — zero hits required.
2. Mock server: serves MusicBrainz artist-search/lookup and CAA cover responses for the
   synthetic artists (deterministic: top ~30 artists matched with 2-3 genres from a fixed
   list, some unmatched, one 503-then-success artist to exercise backoff); runs on an
   ephemeral port; the env override points enrichment at it.
3. Specs (serial where stateful, one worker):
   1. `onboarding.spec`: token gate (wrong token → instruction page), empty dashboard CTA.
   2. `import.spec`: wizard for spotify ZIP (progress → report numbers vs manifest),
      apple CSV, ytm ja JSON (affix behavior visible in library), duplicate re-upload 409
      dialog, wrong-source didYouMean flow, delete an import.
   3. `timeline.spec`: rows/gaps render, metric toggle URL sync, month expansion tables,
      chapter create via drag + edit + overlap lanes + delete.
   4. `trends.spec`: streamgraph renders (path count), legend toggle, genre mode disabled
      pre-enrichment.
   5. `wrapped.spec`: year chips, all 8 cards, print stylesheet applied
      (`page.emulateMedia({ media: 'print' })` nav hidden).
   6. `enrichment.spec`: consent dialog → enable → runner progresses (status polling) →
      auto-pause NOT triggered by the scripted 503 (backoff succeeds) → genre mode shows
      real series in trends → genre map renders nodes/edges, side panel artists, unknown
      chip → artwork phase fetches mock covers and a wrapped top-artist thumbnail renders
      (issue 27 is a dependency); disable → map still renders cached data.
   7. `a11y.spec`: axe-core (`@axe-core/playwright` devDep) on the 7 top-level nav pages
      (`/wrapped/:year` is covered via the Wrapped page) with data present: zero `critical`
      violations (serious+ logged as warnings, tracked but not failing v1).
   8. `security-smoke.spec`: fetch `/api/overview` without token from the page context →
      401; `Host: evil.com` request via API context → 403 (deeper matrix lives in issue 31).
4. Screenshots on failure + trace retain-on-failure; total suite ≤ 10 min on CI.
5. CI: new `e2e` job (ubuntu) after `check`, installing Playwright browsers (chromium only),
   SHA-pinned; artifacts uploaded on failure.

## Acceptance Criteria

- [ ] Full suite green locally (macOS) and in CI (ubuntu, chromium).
- [ ] No real network egress: the issue-25 seam rejects unmapped hosts while active
      (egress guard), plus a final mock-log assertion that every enrichment request arrived
      at the mock server.
- [ ] Import report numbers assert against `manifest.json` (not hardcoded).
- [ ] Token-hygiene teardown: no artifact contains the token substring (asserted).
- [ ] Flake policy: `retries: 1` in CI, 0 locally; any spec needing retry twice in a row is
      a bug (documented in e2e README).
- [ ] Suite runtime ≤ 10 min in CI (measured in PR).

## Validation

`npm run e2e` locally (paste summary), CI run link in PR.

## Dependencies

18, 19, 20, 21, 22, 23, 24, 26, 27, 28 (full journey incl. artwork), 29 (fixtures), 25 (test
seam).

## Non-goals

Cross-browser matrix (chromium only in v1), visual-regression screenshots (v2), load testing
(32), exhaustive security matrix (31).

## Design References

DESIGN.md §16 (E2E row, normative journey), §13 (no-egress rule), ISSUE_PLAN §6.3.
