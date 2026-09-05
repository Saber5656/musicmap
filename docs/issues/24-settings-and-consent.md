# Title

Settings API/UI and enrichment consent flow

## Summary

Implement the settings registry end-to-end per DESIGN.md §17.2/§12.2: GET/PATCH
`/api/settings` with a zod-validated key registry, the `/settings` page (locale, theme,
timezone, default metric), and the enrichment consent section with the full disclosure copy
and enable/disable + delete-enrichment-data actions.

## Context

`enrichment.enabled` is the privacy switch the whole ADR-003 posture hangs on: default OFF,
explicit disclosure before enabling. Timezone changes must trigger the agg rebuild (issue 15).

## Scope

- `src/db/queries/settings.ts` (get/set + first-run tz pinning; `getDisplayTimezone` from
  issue 15 finalized), `src/server/routes/settings.ts`, `src/web/pages/Settings/`,
  `src/web/i18n/{en,ja}/settings.json`.

## Detailed Requirements

1. Query layer: `getSettings(db)` returns the FULL registry with defaults applied (§17.2
   table normative — locale, theme, display.timezone, display.defaultMetric,
   enrichment.enabled; `enrichment.jobState` is internal: excluded from GET, rejected in
   PATCH); `setSetting(db, key, value)` upserts with `updated_at`. First-run timezone
   pinning (normative): on the first `getSettings` call with no `display.timezone` row,
   PERSIST `Intl.DateTimeFormat().resolvedOptions().timeZone` as the value — later OS
   timezone changes must NOT silently shift buckets (only an explicit PATCH does). This
   extends issue 15's read-only `getDisplayTimezone` fallback into a persisted default.
2. Routes:
   - `GET /api/settings` → `{ settings: { locale, theme, "display.timezone",
     "display.defaultMetric", "enrichment.enabled" } }`.
   - `PATCH /api/settings` body `{ key, value }` single-key (simple, atomic); zod
     discriminated validation per key (tz validated by attempting
     `Intl.DateTimeFormat(undefined, { timeZone })` in a try/catch → 400 on throw); unknown
     key → 400 `ERR_VALIDATION`.
   - Side effects in the route: `display.timezone` change → `ensureAggFresh(db, newTz)`
     BEFORE replying (may take seconds on big libraries; acceptable; reply after);
     `enrichment.enabled=false` → signal runner pause (issue 26 decoration if registered —
     optional dependency: use an app decoration hook `mmOnSettingChanged(key,value)` that 26
     subscribes to; define the decoration here as a no-op registry).
   - `GET /api/enrichment/delete-summary` → `{ artistEnrichment, artistGenres, genres,
     artworkRows, artworkFiles }` (row counts + artwork files on disk) — feeds the
     confirmation UI.
   - `POST /api/enrichment/delete-data`: truncates `artist_enrichment`, `artist_genres`,
     `genres`, `artwork` rows and deletes artwork files; only when
     `enrichment.enabled=false` (409 otherwise); 204; idempotent. File-deletion safety
     (T10/T13): resolve `paths(dataDir).artworkDir`, delete only DIRECT regular files in it
     (lstat each; skip symlinks and subdirectories), keep the directory itself. (Placed
     here because it is a consent-surface action; runner not required.)
3. Settings page sections:
   1. Appearance: theme radio (auto/light/dark — replaces issue 17's localStorage fallback
      with server persistence; migrate localStorage value once then remove), locale radio
      (auto/en/ja; applies instantly via i18next.changeLanguage).
   2. Display: timezone select (curated IANA list: all `Intl.supportedValuesOf('timeZone')`
      grouped by region, searchable input), default metric radio. Timezone change shows a
      "recalculating aggregates…" inline spinner until PATCH resolves.
   3. Enrichment (consent block — §12.2 lists the normative disclosure POINTS; write exact
      en+ja copy covering every point): what is sent (artist names, album titles), to whom
      (MusicBrainz / Cover Art Archive links), what is never sent (play history,
      timestamps, counts), caching/retention, "you can delete fetched data anytime".
      Toggle switch → PATCH; when turning ON first time, a confirm dialog restates the
      disclosure with explicit Enable button (not a silent toggle).
      Disabled-state extras: "Delete enrichment data" danger button showing the
      delete-summary counts; typed confirmation — the button stays disabled until the user
      types the literal keyword `DELETE` (same convention as import deletion, issue 18);
      cancel never calls the API.
4. All strings i18n (`src/web/i18n/{en,ja}/settings.json`); consent copy reviewed for
   accuracy against ADR-003 (checklist item).

## Acceptance Criteria

- [ ] Contract tests: GET returns defaults on empty DB; PATCH each key happy+invalid
      (bad tz, bad enum, unknown key, `enrichment.jobState` rejected); tz PATCH calls
      rebuild (spy) and GET reflects persistence.
- [ ] delete-summary returns exact counts (contract test on seeded data).
- [ ] delete-data: 409 while enabled; when disabled, tables emptied + direct regular
      artwork files removed while a planted symlink and subdirectory survive untouched
      (fs assert); directory kept; 204; idempotent second call.
- [ ] First-run timezone pinning: first getSettings persists the system tz; changing the
      test process tz afterwards does not change buckets until PATCH.
- [ ] Typed-confirm UI: button disabled until exact `DELETE` typed; cancel makes no API
      call.
- [ ] UI tests: consent dialog required on first enable (toggle without confirm does not
      PATCH); disable → immediate PATCH; locale switch re-renders strings instantly; theme
      persistence migrates localStorage once.
- [ ] Timezone search select renders and PATCHes a chosen zone (jsdom stub for
      supportedValuesOf when absent).
- [ ] en/ja consent copy complete; key-coverage test.

## Validation

`npm test`, lint, typecheck; manual: enable→disable→delete-data flow screenshots (en+ja) in
PR.

## Dependencies

04, 06, 17 (15's `ensureAggFresh` used — 15 precedes in wave order).

## Non-goals

The enrichment runner itself (26), MusicBrainz client (25), per-event timezone handling (v2).

## Design References

DESIGN.md §17.2 (registry), §12.2 (consent, normative), §8.2/§8.4 (tz rebuild), §9
(endpoints); ADR-003.
