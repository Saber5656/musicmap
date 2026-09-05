# Title

Timeline autobiography view and `/api/timeline` route

## Summary

Implement the timeline route (thin over issue 15's `getTimeline`) and the `/timeline` page per
DESIGN.md §11.1: chronological month rows under sticky year headers, metric bars, top-artist
chips, expandable month detail, zero-month gap markers, and deep-linkable range — WITHOUT the
chapters rail (issue 21 adds it on top).

## Context

This is the flagship "自分史" surface. Layout must leave the left rail gutter reserved so
issue 21 can mount chapters without relayout.

## Scope

- Route in `src/server/routes/stats.ts`; `src/web/pages/Timeline/` components;
  `i18n/{en,ja}/timeline.json`.

## Detailed Requirements

1. Route `GET /api/timeline?bucket=month&from&to&topN&metric`:
   - zod query validation: `bucket` must be the literal `month` (anything else → 400
     `ERR_VALIDATION`); `metric` ∈ `plays|minutes`, default `plays`; `from`/`to` =
     `YYYY-MM`; `topN` 1-10 default 3 — BUT the page always requests `topN=10` and slices
     client-side to 3 for chips, so the expandable detail needs no second fetch, per §11.1.
     Reply schema per §9 example.
2. Page structure:
   - Uses shared `DateRangePicker` + `MetricToggle` (URL-synced; §11.1 deep links).
   - Vertical list, oldest month first; sticky year header rows showing year totals
     (client-side sum of its months: plays, minutes, distinct top artist of the year from
     loaded data). No source-coverage strip here — source coverage lives on the Overview
     page only (keeps this issue independent of issue 19).
   - Month row: `YYYY-MM` label (localized month name), horizontal metric bar scaled to the
     max month within the current range (not global), top-3 artist chips (name + value),
     top-track line (title — artist). Zero months: thin dashed gap row (no bar).
   - Click month row ⇄ inline expansion: two 10-row tables (top artists, top tracks) from
     the same payload's `topArtists`/`topTracks` arrays (§9) — no second fetch. The month
     row's top-track line renders `topTracks[0]`.
   - durationCoverage badge on month rows < 100% when metric = minutes.
   - Left gutter: fixed 96px column rendered empty (issue 21 mounts here).
3. Rendering performance: "All" covers the full imported data span — the expected v1 scale
   is up to ~15 years (~180 month rows; DESIGN §1.1), still plain DOM with no
   virtualization; expansion state is per-month local state; memoize rows.
4. Empty range / no data: EmptyState with import CTA; loading skeleton mimics rows.
5. Keyboard: month rows focusable, Enter toggles expansion; sticky year headers use
   `role="heading"` with `aria-level={2}`.

## Acceptance Criteria

- [ ] Route contract test: golden numbers (mini fixture) incl. the `topTracks` array; zod
      rejects bad `from` format, `bucket=year`, `metric=hours` (each 400 ERR_VALIDATION);
      401 without token; 403 with hostile Host; no token value in captured logs.
- [ ] Component tests: renders fixture payload — correct row count incl. zero-month gap row,
      bar widths proportional (style assertion on 2 known months), chips slice top-3,
      expansion shows 10-row tables without network (apiFetch spy call-count = 1).
- [ ] URL round-trip: `/timeline?from=2019-01&to=2019-12&metric=minutes` restores state.
- [ ] Sticky year header appears once per year with correct totals (fixture assertion).
- [ ] en/ja key coverage; light/dark screenshots in PR.

## Validation

`npm test`, lint, typecheck; manual scroll-through screenshot (en, dark) on the mini fixture.

## Dependencies

15-stats-core, 17-web-shell-i18n.

## Non-goals

Chapters (21), month-level drill pages, virtualization, artwork in rows (v2).

## Design References

DESIGN.md §11.1 (normative), §9 (timeline payload — amended here), §8 (metric semantics).
