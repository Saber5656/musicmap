# Title

Overview dashboard page and `/api/overview` route

## Summary

Implement `GET /api/overview` (thin route over issue 15's `getOverview`) and the `/` dashboard
page: library totals, per-source coverage strips with gap visibility, enrichment summary card,
and first-run empty state — per DESIGN.md §10.1 and §9.

## Context

The dashboard is the landing surface: it must orient a returning user (what data exists, where
the gaps are) and funnel a new user to `/import`.

## Scope

- `src/server/routes/stats.ts` (overview route only; later view routes join this file),
  `src/web/pages/Overview/`, `src/web/i18n/{en,ja}/common.json` additions.

## Detailed Requirements

1. Route: `GET /api/overview` → issue 15 payload verbatim (incl. the zero-month
   `monthlyCounts` contract and `enrichment.enabled` default-false resolution defined
   there); zod reply schema; auth applies; timezone via issue 15's `getDisplayTimezone(db)`
   (shared helper — do not create another).
2. Page cards:
   - Totals hero: plays, minutes (+durationCoveragePct badge when < 100), distinct
     artists/tracks, data range (localized dates).
   - Per-source strip (one per source with data): source name, range, monthly bar sparkline
     from `monthlyCounts` (pure SVG rects, no d3 yet) where zero-months render as visible
     gaps; tooltip month+plays.
   - Enrichment card: disabled → short pitch + link to `/settings`; enabled → matched/pending
     counts + link to `/map`.
   - Quick links: `/timeline`, `/trends`, `/wrapped`, `/map`.
3. First-run (zero events): full-page EmptyState with import CTA (per §14 empty-library row);
   sources with no data simply omit their strip, and a muted "add another source" hint lists
   the missing ones from the fixed universe `spotify` / `apple_music` / `youtube_music`
   (localized labels from i18n).
4. Auto-refresh: refetch on window focus (simple `visibilitychange` hook) so post-import
   returns are fresh.

## Acceptance Criteria

- [ ] Route contract test (inject): payload matches zod schema + golden numbers on the mini
      fixture (incl. zero-month rows inside a source's range); 401 without token; 403 with
      hostile Host; §13.3 headers present (proves the route is registered inside the
      app-factory hook chain).
- [ ] Component tests: empty state (CTA navigates), populated state (3 strips, gap rendering
      for the fixture's empty month), coverage badge visibility logic (100% hides).
- [ ] Sparkline is keyboard/tooltip accessible (`title` per bar acceptable).
- [ ] en/ja bundles complete for the page (key-coverage test).

## Validation

`npm test`, lint, typecheck; manual screenshot (en+ja, light+dark) in PR against the mini
fixture.

## Dependencies

15-stats-core, 17-web-shell-i18n.

## Non-goals

Timeline/trends content, settings UI (24), enrichment actions (26), d3 usage.

## Design References

DESIGN.md §10.1 (/), §9 (/api/overview), §8.1 (source coverage), §13.2-13.3 (route security
inheritance), §14 (empty library).
